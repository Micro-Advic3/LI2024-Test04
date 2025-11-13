"""
Django REST Framework Serializers for Tableau Models
Clean, professional serializers with proper field selection for API responses.
"""

from rest_framework import serializers
from .models import Site, User, Project, Workbook, View, Group, HttpRequest


class SiteSerializer(serializers.ModelSerializer):
    """Serializer for Tableau Site with essential fields for KPI cards."""

    class Meta:
        model = Site
        fields = [
            "id",
            "name",
            "url_namespace",
            "status",
            "luid",
            "created_at",
            "updated_at",
            "storage_quota",
            "subscriptions_enabled",
            "metrics_enabled",
            "flows_enabled",
            "commenting_enabled",
            "time_zone",
        ]
        read_only_fields = fields


class UserSerializer(serializers.ModelSerializer):
    """Serializer for Tableau User with site relationship."""

    site_name = serializers.CharField(source="site.name", read_only=True)

    class Meta:
        model = User
        fields = [
            "id",
            "site",
            "site_name",
            "luid",
            "system_user_id",
            "site_role_id",
            "login_at",
            "created_at",
            "updated_at",
            "system_admin_auto",
        ]
        read_only_fields = fields


class ProjectSerializer(serializers.ModelSerializer):
    """Serializer for Tableau Project with hierarchical info."""

    site_name = serializers.CharField(source="site.name", read_only=True)
    owner_id = serializers.IntegerField(source="owner.id", read_only=True)
    has_children = serializers.SerializerMethodField()

    class Meta:
        model = Project
        fields = [
            "id",
            "name",
            "luid",
            "site",
            "site_name",
            "owner_id",
            "parent_project",
            "has_children",
            "state",
            "description",
            "special",
            "controlled_permissions_enabled",
            "admin_insights_enabled",
            "created_at",
            "updated_at",
        ]
        read_only_fields = fields

    def get_has_children(self, obj):
        """Check if project has child projects."""
        return obj.child_projects.exists()


class WorkbookListSerializer(serializers.ModelSerializer):
    """Lightweight serializer for workbook lists."""

    site_name = serializers.CharField(source="site.name", read_only=True)
    project_name = serializers.CharField(source="project.name", read_only=True)
    owner_id = serializers.IntegerField(source="owner.id", read_only=True)
    view_count_actual = serializers.IntegerField(source="views.count", read_only=True)

    class Meta:
        model = Workbook
        fields = [
            "id",
            "name",
            "luid",
            "repository_url",
            "site",
            "site_name",
            "project",
            "project_name",
            "owner_id",
            "view_count",
            "view_count_actual",
            "size",
            "revision",
            "is_deleted",
            "is_private",
            "created_at",
            "updated_at",
            "last_published_at",
        ]
        read_only_fields = fields


class WorkbookDetailSerializer(WorkbookListSerializer):
    """Detailed serializer for single workbook with extract info."""

    class Meta(WorkbookListSerializer.Meta):
        fields = WorkbookListSerializer.Meta.fields + [
            "document_version",
            "content_version",
            "refreshable_extracts",
            "data_engine_extracts",
            "incrementable_extracts",
            "extracts_refreshed_at",
            "extract_encryption_state",
            "display_tabs",
            "first_published_at",
            "published_all_sheets",
        ]


class ViewSerializer(serializers.ModelSerializer):
    """Serializer for Tableau View/Dashboard."""

    workbook_name = serializers.CharField(source="workbook.name", read_only=True)
    site_name = serializers.CharField(source="site.name", read_only=True)
    owner_id = serializers.IntegerField(source="owner.id", read_only=True)

    class Meta:
        model = View
        fields = [
            "id",
            "name",
            "luid",
            "repository_url",
            "workbook",
            "workbook_name",
            "site",
            "site_name",
            "owner_id",
            "index",
            "title",
            "caption",
            "state",
            "sheettype",
            "revision",
            "is_deleted",
            "created_at",
            "updated_at",
            "first_published_at",
        ]
        read_only_fields = fields


class GroupSerializer(serializers.ModelSerializer):
    """Serializer for Tableau Group."""

    site_name = serializers.CharField(source="site.name", read_only=True)
    owner_id = serializers.IntegerField(source="owner.id", read_only=True)

    class Meta:
        model = Group
        fields = [
            "id",
            "name",
            "luid",
            "site",
            "site_name",
            "owner_id",
            "domain_id",
            "system",
            "minimum_site_role",
            "grant_license_mode",
            "external_user_enabled",
            "last_synchronized",
            "created_at",
            "updated_at",
        ]
        read_only_fields = fields


class HttpRequestSerializer(serializers.ModelSerializer):
    """Serializer for HTTP Request logs - useful for analytics."""

    site_name = serializers.CharField(source="site.name", read_only=True, default=None)
    duration_seconds = serializers.SerializerMethodField()

    class Meta:
        model = HttpRequest
        fields = [
            "id",
            "site",
            "site_name",
            "user_id",
            "controller",
            "action",
            "http_request_uri",
            "status",
            "remote_ip",
            "user_ip",
            "session_id",
            "currentsheet",
            "created_at",
            "completed_at",
            "duration_seconds",
        ]
        read_only_fields = fields

    def get_duration_seconds(self, obj):
        """Calculate request duration in seconds."""
        if obj.completed_at and obj.created_at:
            delta = obj.completed_at - obj.created_at
            return round(delta.total_seconds(), 3)
        return None


# Lightweight serializers for nested relationships
class ViewMinimalSerializer(serializers.ModelSerializer):
    """Minimal view data for embedding in workbook responses."""

    class Meta:
        model = View
        fields = ["id", "name", "luid", "sheettype", "index"]
        read_only_fields = fields


class WorkbookWithViewsSerializer(WorkbookDetailSerializer):
    """Workbook with nested views - useful for dashboard overview."""

    views = ViewMinimalSerializer(many=True, read_only=True)

    class Meta(WorkbookDetailSerializer.Meta):
        fields = WorkbookDetailSerializer.Meta.fields + ["views"]
