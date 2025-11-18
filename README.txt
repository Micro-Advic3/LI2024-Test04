class UnderscoreView(models.Model):
    """
    Tableau's _views table (with underscore) - contains URL-friendly names
    """
    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    view_url = models.TextField()  # THIS IS THE KEY FIELD FOR URLS!
    workbook = models.ForeignKey('UnderscoreWorkbook', on_delete=models.CASCADE, db_column='workbook_id')
    index = models.IntegerField()
    site = models.ForeignKey('Site', on_delete=models.CASCADE, db_column='site_id')
    created_at = models.DateTimeField()
    owner_id = models.IntegerField()
    owner_name = models.CharField(max_length=255, null=True, blank=True)
    title = models.TextField(null=True, blank=True)
    caption = models.TextField(null=True, blank=True)
    
    class Meta:
        managed = False
        db_table = '_views'  # Note the underscore!
        
    def __str__(self):
        return self.name


class UnderscoreWorkbook(models.Model):
    """
    Tableau's _workbooks table (with underscore) - contains URL-friendly names
    """
    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    workbook_url = models.TextField()  # THIS IS THE KEY FIELD FOR URLS!
    site = models.ForeignKey('Site', on_delete=models.CASCADE, db_column='site_id')
    project_id = models.IntegerField()
    project_name = models.CharField(max_length=255, null=True, blank=True)
    owner_id = models.IntegerField()
    owner_name = models.CharField(max_length=255, null=True, blank=True)
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    view_count = models.IntegerField(null=True, blank=True)
    size = models.BigIntegerField(null=True, blank=True)
    
    class Meta:
        managed = False
        db_table = '_workbooks'  # Note the underscore!
        
    def __str__(self):
        return self.name



************************************************************************************************************************


from .models import UnderscoreView, UnderscoreWorkbook, Site

@action(detail=False, methods=['get'])
def explore(self, request):
    """
    FINAL VERSION - Using _views and _workbooks tables with URL-friendly names!
    """
    
    search = request.query_params.get('search', '')
    limit = int(request.query_params.get('limit', 100))
    
    # Query the CORRECT tables (with underscores!)
    queryset = UnderscoreView.objects.select_related(
        'workbook',
        'site'
    ).all()
    
    # Apply search if provided
    if search:
        queryset = queryset.filter(
            models.Q(name__icontains=search) |
            models.Q(workbook__name__icontains=search)
        )
    
    queryset = queryset.order_by('name')[:limit]
    
    results = []
    for view in queryset:
        # Build URL using the CORRECT fields!
        tableau_url = f"https://tableau.cib.echonet/#/site/{view.site.url_namespace}/views/{view.workbook.workbook_url}/{view.view_url}?:iid={view.index}"
        
        results.append({
            'id': view.id,
            'view_name': view.name,
            'view_url': view.view_url,  # URL-friendly name!
            'view_index': view.index,
            'workbook_name': view.workbook.name,
            'workbook_url': view.workbook.workbook_url,  # URL-friendly name!
            'site_name': view.site.name,
            'site_url_namespace': view.site.url_namespace,
            'tableau_url': tableau_url,  # CORRECT URL!
        })
    
    return Response({
        'count': len(results),
        'results': results
    })

