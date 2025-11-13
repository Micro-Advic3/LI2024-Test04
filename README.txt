"""
Django REST Framework ViewSets for Tableau API
Professional read-only API with filtering, search, and pagination.
"""

from rest_framework import viewsets, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Count, Avg, Q
from django.utils import timezone
from datetime import timedelta

from django.db.models import Count, Q, F, Value, CharField
from django.db.models.functions import Concat
from datetime import datetime, timedelta
from django.utils import timezone


from .models import Site, User, Project, Workbook, View, Group, HttpRequest
from .serializers import (
    SiteSerializer,
    UserSerializer,
    ProjectSerializer,
    WorkbookListSerializer,
    WorkbookDetailSerializer,
    WorkbookWithViewsSerializer,
    ViewSerializer,
    GroupSerializer,
    HttpRequestSerializer,
)


class BaseTableauViewSet(viewsets.ReadOnlyModelViewSet):
    """
    Base ViewSet for all Tableau models.
    Read-only by default since we don't want to modify Tableau data.
    """

    filter_backends = [filters.SearchFilter, filters.OrderingFilter]

    def get_queryset(self):
        """Override to add prefetch_related/select_related for performance."""
        return super().get_queryset()


class SiteViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau Sites.

    Endpoints:
    - GET /api/tableau/sites/ - List all sites
    - GET /api/tableau/sites/{id}/ - Get site details
    - GET /api/tableau/sites/{id}/stats/ - Get site statistics
    """

    queryset = Site.objects.all()
    serializer_class = SiteSerializer
    search_fields = ["name", "url_namespace"]
    ordering_fields = ["created_at", "name"]
    ordering = ["-created_at"]

    @action(detail=True, methods=["get"])
    def stats(self, request, pk=None):
        """Get statistics for a specific site."""
        site = self.get_object()

        stats = {
            "site_id": site.id,
            "site_name": site.name,
            "total_users": site.users.count(),
            "total_workbooks": site.workbooks.filter(is_deleted=False).count(),
            "total_views": site.views.filter(is_deleted=False).count(),
            "total_projects": site.projects.count(),
            "total_groups": site.groups.count(),
            "active_workbooks": site.workbooks.filter(
                is_deleted=False,
                last_published_at__gte=timezone.now() - timedelta(days=30),
            ).count(),
        }

        return Response(stats)


class UserViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau Users.

    Endpoints:
    - GET /api/tableau/users/ - List all users
    - GET /api/tableau/users/{id}/ - Get user details

    Filters:
    - ?site={site_id} - Filter by site
    - ?search={term} - Search by system_user_id
    """

    queryset = User.objects.select_related("site").all()
    serializer_class = UserSerializer
    search_fields = ["system_user_id"]
    ordering_fields = ["created_at", "login_at"]
    ordering = ["-created_at"]
    filterset_fields = ["site", "site_role_id", "system_admin_auto"]


class ProjectViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau Projects.

    Endpoints:
    - GET /api/tableau/projects/ - List all projects
    - GET /api/tableau/projects/{id}/ - Get project details
    - GET /api/tableau/projects/{id}/workbooks/ - Get project workbooks

    Filters:
    - ?site={site_id} - Filter by site
    - ?parent_project={project_id} - Get child projects
    - ?state=active - Filter by state
    """

    queryset = Project.objects.select_related("site", "owner").all()
    serializer_class = ProjectSerializer
    search_fields = ["name", "description"]
    ordering_fields = ["created_at", "name"]
    ordering = ["name"]
    filterset_fields = ["site", "owner", "parent_project", "state", "special"]

    @action(detail=True, methods=["get"])
    def workbooks(self, request, pk=None):
        """Get all workbooks in this project."""
        project = self.get_object()
        workbooks = project.workbooks.filter(is_deleted=False)
        serializer = WorkbookListSerializer(workbooks, many=True)
        return Response(serializer.data)


class WorkbookViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau Workbooks.

    Endpoints:
    - GET /api/tableau/workbooks/ - List all workbooks
    - GET /api/tableau/workbooks/{id}/ - Get workbook details
    - GET /api/tableau/workbooks/{id}/views/ - Get workbook views

    Filters:
    - ?site={site_id} - Filter by site
    - ?project={project_id} - Filter by project
    - ?owner={user_id} - Filter by owner
    - ?is_deleted=false - Exclude deleted workbooks
    """

    queryset = Workbook.objects.select_related("site", "owner", "project").all()
    search_fields = ["name", "repository_url"]
    ordering_fields = ["created_at", "updated_at", "name", "size"]
    ordering = ["-updated_at"]
    filterset_fields = ["site", "project", "owner", "is_deleted", "is_private"]

    def get_serializer_class(self):
        """Use detailed serializer for retrieve, list serializer for list."""
        if self.action == "retrieve":
            return WorkbookDetailSerializer
        elif self.action == "with_views":
            return WorkbookWithViewsSerializer
        return WorkbookListSerializer

    @action(detail=True, methods=["get"])
    def views(self, request, pk=None):
        """Get all views/dashboards in this workbook."""
        workbook = self.get_object()
        views = workbook.views.filter(is_deleted=False).order_by("index")
        serializer = ViewSerializer(views, many=True)
        return Response(serializer.data)

    @action(detail=False, methods=["get"])
    def recent(self, request):
        """Get recently updated workbooks."""
        days = int(request.query_params.get("days", 7))
        cutoff_date = timezone.now() - timedelta(days=days)

        workbooks = self.queryset.filter(
            is_deleted=False, updated_at__gte=cutoff_date
        ).order_by("-updated_at")[:50]

        serializer = self.get_serializer(workbooks, many=True)
        return Response(serializer.data)


class ViewViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau Views/Dashboards.

    Endpoints:
    - GET /api/tableau/views/ - List all views
    - GET /api/tableau/views/{id}/ - Get view details

    Filters:
    - ?workbook={workbook_id} - Filter by workbook
    - ?site={site_id} - Filter by site
    - ?sheettype=dashboard - Filter by type
    """

    queryset = View.objects.select_related("workbook", "site", "owner").all()
    serializer_class = ViewSerializer
    search_fields = ["name", "title"]
    ordering_fields = ["created_at", "updated_at", "name", "index"]
    ordering = ["workbook", "index"]
    filterset_fields = ["workbook", "site", "owner", "state", "sheettype", "is_deleted"]


class GroupViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau Groups.

    Endpoints:
    - GET /api/tableau/groups/ - List all groups
    - GET /api/tableau/groups/{id}/ - Get group details

    Filters:
    - ?site={site_id} - Filter by site
    - ?system=true - Filter system groups
    """

    queryset = Group.objects.select_related("site", "owner").all()
    serializer_class = GroupSerializer
    search_fields = ["name"]
    ordering_fields = ["created_at", "name"]
    ordering = ["name"]
    filterset_fields = ["site", "owner", "system", "domain_id"]


class HttpRequestViewSet(BaseTableauViewSet):
    """
    API endpoint for Tableau HTTP Request Logs.

    Endpoints:
    - GET /api/tableau/requests/ - List all requests
    - GET /api/tableau/requests/{id}/ - Get request details
    - GET /api/tableau/requests/analytics/ - Get request analytics

    Filters:
    - ?site={site_id} - Filter by site
    - ?status=200 - Filter by HTTP status
    - ?controller={name} - Filter by controller
    """

    queryset = HttpRequest.objects.select_related("site").all()
    serializer_class = HttpRequestSerializer
    search_fields = ["controller", "action", "http_request_uri"]
    ordering_fields = ["created_at", "status"]
    ordering = ["-created_at"]
    filterset_fields = ["site", "user_id", "status", "controller"]

    @action(detail=False, methods=["get"])
    def analytics(self, request):
        """Get HTTP request analytics for the last 24 hours."""
        cutoff_time = timezone.now() - timedelta(hours=24)

        requests_qs = self.queryset.filter(created_at__gte=cutoff_time)

        analytics = {
            "total_requests": requests_qs.count(),
            "unique_users": requests_qs.exclude(user_id__isnull=True)
            .values("user_id")
            .distinct()
            .count(),
            "status_breakdown": {
                "2xx": requests_qs.filter(status__gte=200, status__lt=300).count(),
                "3xx": requests_qs.filter(status__gte=300, status__lt=400).count(),
                "4xx": requests_qs.filter(status__gte=400, status__lt=500).count(),
                "5xx": requests_qs.filter(status__gte=500, status__lt=600).count(),
            },
            "top_controllers": list(
                requests_qs.values("controller")
                .annotate(count=Count("id"))
                .order_by("-count")[:10]
            ),
            "top_actions": list(
                requests_qs.values("action")
                .annotate(count=Count("id"))
                .order_by("-count")[:10]
            ),
        }

        return Response(analytics)


class InsightsViewSet(viewsets.ViewSet):
    """
    Insights Page Exploration Endpoints
    Help you discover what data to show!
    """

    @action(detail=False, methods=["get"])
    def explore(self, request):
        """
        Explore ALL available dashboard data with joins.
        This shows you everything so you can decide what to display!

        Query params:
        - ?tag={project_name} - Filter by project
        - ?limit=20 - Limit results (default 50)
        """
        limit = int(request.query_params.get("limit", 50))
        tag_filter = request.query_params.get("tag", None)

        # Get all views (dashboards) with ALL related data
        queryset = View.objects.select_related(
            "workbook", "workbook__project", "workbook__site", "workbook__owner", "site"
        ).filter(
            is_deleted=False,
            workbook__is_deleted=False,
            sheettype="dashboard",  # Only dashboards
        )

        # Filter by project if requested
        if tag_filter:
            queryset = queryset.filter(workbook__project__name__icontains=tag_filter)

        # Order by most recently updated
        queryset = queryset.order_by("-updated_at")[:limit]

        # Build response with ALL data
        results = []
        for view in queryset:
            # Calculate last refresh time
            time_diff = timezone.now() - view.updated_at
            hours_ago = int(time_diff.total_seconds() / 3600)
            if hours_ago < 24:
                last_refresh = f"{hours_ago}h ago"
            else:
                days_ago = int(hours_ago / 24)
                last_refresh = f"{days_ago}d ago"

            # Get view count from http_requests (last 30 days)
            thirty_days_ago = timezone.now() - timedelta(days=30)
            view_count = HttpRequest.objects.filter(
                currentsheet=view.name, created_at__gte=thirty_days_ago
            ).count()

            results.append(
                {
                    "id": str(view.id),
                    "view_id": view.id,
                    "view_name": view.name,
                    "view_title": view.title or view.name,
                    "view_luid": str(view.luid),
                    # Workbook info
                    "workbook_id": view.workbook.id,
                    "workbook_name": view.workbook.name,
                    # Project info (could be your "tag")
                    "project_id": view.workbook.project.id,
                    "project_name": view.workbook.project.name,
                    # Site info
                    "site_id": view.site.id,
                    "site_name": view.site.name,
                    # Owner info
                    "owner_id": view.workbook.owner.id,
                    "owner_system_user_id": view.workbook.owner.system_user_id,
                    # Timestamps
                    "created_at": view.created_at,
                    "updated_at": view.updated_at,
                    "last_refresh": last_refresh,
                    "first_published_at": view.first_published_at,
                    # Metadata
                    "sheettype": view.sheettype,
                    "state": view.state,
                    "repository_url": view.repository_url,
                    # Calculated fields
                    "view_count_30_days": view_count,
                    # Potential Tableau URL (you'll need to adjust)
                    "tableau_url_base": f"https://tableau.yourcompany.com/views/{view.workbook.repository_url}/{view.repository_url}",
                }
            )

        return Response(
            {
                "count": len(results),
                "results": results,
                "help": {
                    "message": "This shows ALL available data. Use this to decide what to display on Insights page!",
                    "available_fields": list(results[0].keys()) if results else [],
                    "suggestions": {
                        "title": "Use view_title or view_name",
                        "tag": "Use project_name",
                        "owner": "Use owner_system_user_id (or map to real names)",
                        "view_count": "Use view_count_30_days",
                        "last_refresh": "Already calculated as human-readable",
                    },
                },
            }
        )

    @action(detail=False, methods=["get"])
    def top_views(self, request):
        """
        Get most viewed dashboards (by HTTP requests).
        Perfect for "Popular Dashboards" section!

        Query params:
        - ?days=30 - Time period (default 30 days)
        - ?limit=10 - Number of results (default 10)
        """
        days = int(request.query_params.get("days", 30))
        limit = int(request.query_params.get("limit", 10))

        cutoff_date = timezone.now() - timedelta(days=days)

        # Count views by currentsheet in http_requests
        top_sheets = (
            HttpRequest.objects.filter(created_at__gte=cutoff_date)
            .values("currentsheet")
            .annotate(view_count=Count("id"))
            .order_by("-view_count")[: limit * 2]
        )  # Get more to filter

        # Get actual View objects
        sheet_names = [
            item["currentsheet"] for item in top_sheets if item["currentsheet"]
        ]

        views = View.objects.select_related(
            "workbook", "workbook__project", "workbook__site", "workbook__owner"
        ).filter(name__in=sheet_names, is_deleted=False, sheettype="dashboard")

        # Build results with view counts
        view_count_map = {
            item["currentsheet"]: item["view_count"] for item in top_sheets
        }

        results = []
        for view in views[:limit]:
            results.append(
                {
                    "id": str(view.id),
                    "title": view.title or view.name,
                    "subtitle": f"Last refresh: {view.updated_at.strftime('%Y-%m-%d')}",
                    "tag": view.workbook.project.name,
                    "meta": f"Viewed {view_count_map.get(view.name, 0)} times in last {days} days - Owner: User {view.workbook.owner.system_user_id}",
                    "view_count": view_count_map.get(view.name, 0),
                    "workbook_name": view.workbook.name,
                    "project_name": view.workbook.project.name,
                    "owner_id": view.workbook.owner.system_user_id,
                }
            )

        return Response(
            {"count": len(results), "time_period_days": days, "results": results}
        )

    @action(detail=False, methods=["get"])
    def recent_dashboards(self, request):
        """
        Get recently updated dashboards.
        Perfect for "What's New" section!

        Query params:
        - ?days=7 - Time period (default 7 days)
        - ?limit=10 - Number of results (default 10)
        """
        days = int(request.query_params.get("days", 7))
        limit = int(request.query_params.get("limit", 10))

        cutoff_date = timezone.now() - timedelta(days=days)

        views = (
            View.objects.select_related(
                "workbook", "workbook__project", "workbook__site", "workbook__owner"
            )
            .filter(
                is_deleted=False, sheettype="dashboard", updated_at__gte=cutoff_date
            )
            .order_by("-updated_at")[:limit]
        )

        results = []
        for view in views:
            time_diff = timezone.now() - view.updated_at
            hours_ago = int(time_diff.total_seconds() / 3600)
            if hours_ago < 24:
                last_refresh = f"{hours_ago}h ago"
            else:
                days_ago = int(hours_ago / 24)
                last_refresh = f"{days_ago}d ago"

            results.append(
                {
                    "id": str(view.id),
                    "title": view.title or view.name,
                    "subtitle": f"Validated - Last refresh: {last_refresh}",
                    "tag": view.workbook.project.name,
                    "meta": f"Owner: User {view.workbook.owner.system_user_id}",
                    "updated_at": view.updated_at,
                    "workbook_name": view.workbook.name,
                    "project_name": view.workbook.project.name,
                }
            )

        return Response(
            {"count": len(results), "time_period_days": days, "results": results}
        )

    @action(detail=False, methods=["get"])
    def by_project(self, request):
        """
        Group dashboards by project (tag).
        See which projects have the most dashboards!
        """
        projects_data = (
            Project.objects.annotate(
                dashboard_count=Count(
                    "workbooks__views",
                    filter=Q(
                        workbooks__views__sheettype="dashboard",
                        workbooks__views__is_deleted=False,
                        workbooks__is_deleted=False,
                    ),
                )
            )
            .filter(dashboard_count__gt=0)
            .order_by("-dashboard_count")
        )

        results = []
        for project in projects_data:
            results.append(
                {
                    "project_id": project.id,
                    "project_name": project.name,
                    "dashboard_count": project.dashboard_count,
                    "site_name": project.site.name,
                }
            )

        return Response({"count": len(results), "results": results})
