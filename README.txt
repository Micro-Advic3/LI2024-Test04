"""
Django Admin Configuration for Tableau Models
Read-only admin interface for browsing Tableau data.
"""

from django.contrib import admin
from .models import Site, User, Project, Workbook, View, Group, HttpRequest


@admin.register(Site)
class SiteAdmin(admin.ModelAdmin):
    """Admin interface for Sites."""

    list_display = ["id", "name", "url_namespace", "status", "created_at"]
    list_filter = ["status", "subscriptions_enabled", "metrics_enabled"]
    search_fields = ["name", "url_namespace", "luid"]
    readonly_fields = [field.name for field in Site._meta.fields]

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False


@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    """Admin interface for Users."""

    list_display = ["id", "site", "system_user_id", "site_role_id", "login_at"]
    list_filter = ["site", "system_admin_auto"]
    search_fields = ["luid", "system_user_id"]
    readonly_fields = [field.name for field in User._meta.fields]

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False


@admin.register(Project)
class ProjectAdmin(admin.ModelAdmin):
    """Admin interface for Projects."""

    list_display = ["id", "name", "site", "owner", "state", "special"]
    list_filter = ["site", "state", "special"]
    search_fields = ["name", "luid", "description"]
    readonly_fields = [field.name for field in Project._meta.fields]

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False


@admin.register(Workbook)
class WorkbookAdmin(admin.ModelAdmin):
    """Admin interface for Workbooks."""

    list_display = [
        "id",
        "name",
        "site",
        "project",
        "owner",
        "view_count",
        "is_deleted",
    ]
    list_filter = ["site", "project", "is_deleted", "is_private"]
    search_fields = ["name", "repository_url", "luid"]
    readonly_fields = [field.name for field in Workbook._meta.fields]

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False


@admin.register(View)
class ViewAdmin(admin.ModelAdmin):
    """Admin interface for Views."""

    list_display = ["id", "name", "workbook", "sheettype", "state", "is_deleted"]
    list_filter = ["site", "workbook", "sheettype", "state", "is_deleted"]
    search_fields = ["name", "title", "luid"]
    readonly_fields = [field.name for field in View._meta.fields]

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False


@admin.register(Group)
class GroupAdmin(admin.ModelAdmin):
    """Admin interface for Groups."""

    list_display = ["id", "name", "site", "owner", "system", "domain_id"]
    list_filter = ["site", "system"]
    search_fields = ["name", "luid"]
    readonly_fields = [field.name for field in Group._meta.fields]

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False


@admin.register(HttpRequest)
class HttpRequestAdmin(admin.ModelAdmin):
    """Admin interface for HTTP Requests."""

    list_display = ["id", "site", "controller", "action", "status", "created_at"]
    list_filter = ["site", "status", "controller"]
    search_fields = ["action", "http_request_uri", "remote_ip"]
    readonly_fields = [field.name for field in HttpRequest._meta.fields]
    date_hierarchy = "created_at"

    def has_add_permission(self, request):
        return False

    def has_delete_permission(self, request, obj=None):
        return False
