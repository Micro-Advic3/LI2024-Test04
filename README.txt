"""
Tableau Server Database ORM Models
Professional implementation with complete field mappings for all Tableau tables.
Read-only models for existing Tableau Server database.
"""

from django.db import models
from django.utils.translation import gettext_lazy as _


class BaseTableauModel(models.Model):
    """Abstract base model for all Tableau models with common patterns."""

    class Meta:
        abstract = True
        managed = False  # Don't let Django manage these tables

    def __str__(self):
        return f"{self.__class__.__name__}({self.pk})"


class Site(BaseTableauModel):
    """
    Tableau Site - Multi-tenancy hub.
    Each site is an isolated environment with its own users, workbooks, etc.
    """

    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255, help_text="Site name")
    url_namespace = models.CharField(
        max_length=255, help_text="URL identifier for site"
    )
    status = models.CharField(max_length=50, help_text="active/suspended/locked")
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()

    # Capacity & Quotas
    storage_quota = models.BigIntegerField(
        null=True, blank=True, help_text="Max storage in MB"
    )
    content_admin_mode = models.IntegerField(null=True, blank=True)

    # Feature Flags (selective - add more as needed)
    subscriptions_enabled = models.BooleanField(default=False)
    authoring_disabled = models.BooleanField(default=False)
    sheet_image_enabled = models.BooleanField(default=False)
    guest_access_enabled = models.BooleanField(default=False)
    commenting_enabled = models.BooleanField(default=False)
    flows_enabled = models.BooleanField(default=False)
    metrics_enabled = models.BooleanField(default=False)

    # System fields
    luid = models.UUIDField(unique=True, help_text="Immutable UUID identifier")
    lock_version = models.IntegerField(default=0, help_text="Optimistic locking")
    time_zone = models.TextField(null=True, blank=True, help_text="IANA timezone")

    class Meta(BaseTableauModel.Meta):
        db_table = "sites"
        verbose_name = _("Site")
        verbose_name_plural = _("Sites")

    def __str__(self):
        return self.name


class User(BaseTableauModel):
    """
    Tableau User - Links system users to sites.
    Site-specific user settings and permissions.
    """

    id = models.IntegerField(primary_key=True)
    site = models.ForeignKey(
        Site, on_delete=models.DO_NOTHING, db_column="site_id", related_name="users"
    )
    system_user_id = models.IntegerField(help_text="FK to system_users table")
    login_at = models.DateTimeField(null=True, blank=True, help_text="Last login")
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    site_role_id = models.IntegerField(help_text="User's site role")
    system_admin_auto = models.BooleanField(default=False)
    luid = models.UUIDField(unique=True)
    lock_version = models.IntegerField(default=0)

    class Meta(BaseTableauModel.Meta):
        db_table = "users"
        verbose_name = _("User")
        verbose_name_plural = _("Users")
        indexes = [
            models.Index(fields=["site", "system_user_id"]),
        ]

    def __str__(self):
        return f"User {self.id} on {self.site.name}"


class Project(BaseTableauModel):
    """
    Tableau Project - Organizational container for workbooks.
    Supports hierarchical structure with parent-child relationships.
    """

    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    site = models.ForeignKey(
        Site, on_delete=models.DO_NOTHING, db_column="site_id", related_name="projects"
    )
    owner = models.ForeignKey(
        User,
        on_delete=models.DO_NOTHING,
        db_column="owner_id",
        related_name="owned_projects",
    )
    parent_project = models.ForeignKey(
        "self",
        on_delete=models.DO_NOTHING,
        db_column="parent_project_id",
        null=True,
        blank=True,
        related_name="child_projects",
    )
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    state = models.CharField(max_length=50, help_text="active/inactive")
    description = models.TextField(null=True, blank=True)
    special = models.IntegerField(null=True, blank=True, help_text="1=system project")
    luid = models.UUIDField(unique=True)
    controlled_permissions_enabled = models.BooleanField(default=False)
    admin_insights_enabled = models.BooleanField(default=False)
    nested_projects_permissions_included = models.BooleanField(default=False)
    controlling_permissions_project_id = models.IntegerField(null=True, blank=True)
    lower_name = models.CharField(max_length=255, help_text="Lowercase for indexing")

    class Meta(BaseTableauModel.Meta):
        db_table = "projects"
        verbose_name = _("Project")
        verbose_name_plural = _("Projects")
        indexes = [
            models.Index(fields=["site", "name"]),
        ]

    def __str__(self):
        return f"{self.name} (Project)"


class Workbook(BaseTableauModel):
    """
    Tableau Workbook - Core content object.
    Contains views/dashboards and manages extracts.
    """

    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    repository_url = models.TextField(help_text="Unique ASCII URL-safe name")
    site = models.ForeignKey(
        Site, on_delete=models.DO_NOTHING, db_column="site_id", related_name="workbooks"
    )
    owner = models.ForeignKey(
        User,
        on_delete=models.DO_NOTHING,
        db_column="owner_id",
        related_name="owned_workbooks",
    )
    project = models.ForeignKey(
        Project,
        on_delete=models.DO_NOTHING,
        db_column="project_id",
        related_name="workbooks",
    )
    parent_workbook = models.ForeignKey(
        "self",
        on_delete=models.DO_NOTHING,
        db_column="parent_workbook_id",
        null=True,
        blank=True,
        related_name="derived_workbooks",
    )
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    first_published_at = models.DateTimeField(null=True, blank=True)
    last_published_at = models.DateTimeField(null=True, blank=True)

    # Content metadata
    view_count = models.IntegerField(default=0)
    size = models.BigIntegerField(help_text="Size in bytes")
    revision = models.CharField(max_length=50, help_text="Version: 1.0, 1.1, etc.")
    document_version = models.CharField(max_length=50, null=True, blank=True)
    content_version = models.IntegerField(null=True, blank=True)

    # Extract management
    refreshable_extracts = models.BooleanField(default=False)
    data_engine_extracts = models.BooleanField(default=False)
    incrementable_extracts = models.BooleanField(default=False)
    extracts_refreshed_at = models.DateTimeField(null=True, blank=True)
    extracts_incremented_at = models.DateTimeField(null=True, blank=True)
    extract_encryption_state = models.SmallIntegerField(null=True, blank=True)
    extract_creation_pending = models.SmallIntegerField(null=True, blank=True)
    extract_storage_format = models.SmallIntegerField(null=True, blank=True)

    # UI & Display
    display_tabs = models.BooleanField(default=True)
    default_view_index = models.IntegerField(null=True, blank=True)
    embedded = models.TextField(
        null=True, blank=True, help_text="Encrypted credentials"
    )
    thumb_user = models.IntegerField(null=True, blank=True)

    # System fields
    luid = models.UUIDField(unique=True)
    lock_version = models.IntegerField(default=0)
    is_deleted = models.BooleanField(default=False, help_text="Soft delete flag")
    is_private = models.BooleanField(default=False)
    published_all_sheets = models.BooleanField(default=False)
    modified_by_user_id = models.IntegerField(null=True, blank=True)
    data_id = models.CharField(max_length=255, null=True, blank=True)
    reduced_data_id = models.CharField(max_length=255, null=True, blank=True)
    asset_key_id = models.IntegerField(null=True, blank=True)

    class Meta(BaseTableauModel.Meta):
        db_table = "workbooks"
        verbose_name = _("Workbook")
        verbose_name_plural = _("Workbooks")
        indexes = [
            models.Index(fields=["site", "project"]),
            models.Index(fields=["owner"]),
        ]

    def __str__(self):
        return self.name


class View(BaseTableauModel):
    """
    Tableau View - Individual dashboard/sheet within a workbook.
    Represents visualizations that users interact with.
    """

    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    repository_url = models.TextField(help_text="Unique identifier")
    workbook = models.ForeignKey(
        Workbook,
        on_delete=models.DO_NOTHING,
        db_column="workbook_id",
        related_name="views",
    )
    site = models.ForeignKey(
        Site, on_delete=models.DO_NOTHING, db_column="site_id", related_name="views"
    )
    owner = models.ForeignKey(
        User,
        on_delete=models.DO_NOTHING,
        db_column="owner_id",
        related_name="owned_views",
    )
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    first_published_at = models.DateTimeField(null=True, blank=True)

    # View metadata
    index = models.IntegerField(help_text="Position in workbook")
    description = models.TextField(null=True, blank=True)
    fields = models.TextField(
        null=True, blank=True, help_text="Extracted fields from TWB"
    )
    title = models.TextField(null=True, blank=True)
    caption = models.TextField(null=True, blank=True)
    sheet_id = models.CharField(max_length=255, null=True, blank=True)
    state = models.CharField(max_length=50, help_text="active/disabled")
    sheettype = models.CharField(max_length=50, help_text="story/dashboard/view")
    revision = models.CharField(max_length=50, help_text="Version tracking")
    thumbnail_id = models.CharField(max_length=255, null=True, blank=True)

    # System fields
    luid = models.UUIDField(unique=True)
    is_deleted = models.BooleanField(default=False)

    class Meta(BaseTableauModel.Meta):
        db_table = "views"
        verbose_name = _("View")
        verbose_name_plural = _("Views")
        indexes = [
            models.Index(fields=["workbook", "index"]),
            models.Index(fields=["site"]),
        ]

    def __str__(self):
        return f"{self.name} (View)"


class Group(BaseTableauModel):
    """
    Tableau Group - User grouping for permissions.
    Can be locally created or synced from Active Directory.
    """

    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    site = models.ForeignKey(
        Site, on_delete=models.DO_NOTHING, db_column="site_id", related_name="groups"
    )
    owner = models.ForeignKey(
        User,
        on_delete=models.DO_NOTHING,
        db_column="owner_id",
        related_name="owned_groups",
    )
    domain_id = models.IntegerField(
        null=True, blank=True, help_text="FK to domains (AD)"
    )
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    last_synchronized = models.DateTimeField(
        null=True, blank=True, help_text="AD sync time"
    )

    # Group settings
    system = models.BooleanField(default=False, help_text="Tableau-created group")
    minimum_site_role = models.CharField(max_length=100, null=True, blank=True)
    minimum_site_role_id = models.IntegerField(null=True, blank=True)
    grant_license_mode = models.CharField(
        max_length=50, null=True, blank=True, help_text="ON_SYNC/ON_LOGIN"
    )
    external_user_enabled = models.BooleanField(default=False)

    # System fields
    luid = models.UUIDField(unique=True)

    class Meta(BaseTableauModel.Meta):
        db_table = "groups"
        verbose_name = _("Group")
        verbose_name_plural = _("Groups")
        indexes = [
            models.Index(fields=["site", "name"]),
        ]

    def __str__(self):
        return f"{self.name} (Group)"


class HttpRequest(BaseTableauModel):
    """
    Tableau HTTP Request Log - Tracks all requests to Tableau Server.
    Useful for analytics, auditing, and debugging.
    """

    id = models.BigIntegerField(primary_key=True)
    site = models.ForeignKey(
        Site,
        on_delete=models.DO_NOTHING,
        db_column="site_id",
        null=True,
        blank=True,
        related_name="http_requests",
    )
    user_id = models.IntegerField(null=True, blank=True, help_text="FK to users")

    # Request details
    controller = models.CharField(max_length=255, help_text="Application component")
    action = models.CharField(max_length=255, help_text="Action requested")
    http_referer = models.CharField(max_length=500, null=True, blank=True)
    http_user_agent = models.CharField(max_length=500, null=True, blank=True)
    http_request_uri = models.TextField(help_text="Full URI")
    remote_ip = models.CharField(
        max_length=50, help_text="Client IP (server perspective)"
    )
    user_ip = models.CharField(max_length=50, help_text="Originator IP")
    port = models.IntegerField(help_text="Request port")
    worker = models.CharField(max_length=255, help_text="Worker machine")

    # Session & State
    session_id = models.CharField(max_length=255, null=True, blank=True)
    user_cookie = models.CharField(max_length=500, null=True, blank=True)
    vizql_session = models.TextField(null=True, blank=True)
    currentsheet = models.CharField(max_length=255, null=True, blank=True)

    # Response
    status = models.IntegerField(help_text="HTTP status code")
    flags_used = models.BigIntegerField(help_text="Feature flags bitmap")

    # Timestamps
    created_at = models.DateTimeField(help_text="Request received")
    completed_at = models.DateTimeField(
        null=True, blank=True, help_text="Request completed"
    )

    class Meta(BaseTableauModel.Meta):
        db_table = "http_requests"
        verbose_name = _("HTTP Request")
        verbose_name_plural = _("HTTP Requests")
        indexes = [
            models.Index(fields=["created_at"]),
            models.Index(fields=["site", "created_at"]),
        ]

    def __str__(self):
        return f"Request {self.id}: {self.controller}/{self.action}"
