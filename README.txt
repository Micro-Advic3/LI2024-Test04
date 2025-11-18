class CustomizedView(models.Model):
    """
    Tableau's customized_views table - user-created/accessed views with URL data
    """
    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=255)
    description = models.TextField(null=True, blank=True)
    view = models.ForeignKey('View', on_delete=models.CASCADE, db_column='view_id', related_name='customized_views')
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()
    creator_id = models.IntegerField()
    public = models.BooleanField(default=False)
    size = models.IntegerField(null=True, blank=True)
    site = models.ForeignKey('Site', on_delete=models.CASCADE, db_column='site_id')
    url_id = models.CharField(max_length=255)  # THIS IS THE KEY FIELD!
    start_view = models.ForeignKey('View', on_delete=models.CASCADE, db_column='start_view_id', related_name='starting_customized_views', null=True, blank=True)
    luid = models.UUIDField()
    data_id = models.CharField(max_length=255, null=True, blank=True)
    thumbnail_id = models.CharField(max_length=255, null=True, blank=True)
    hidden = models.BooleanField(default=False)
    created_by_feature = models.IntegerField(null=True, blank=True)
    accessed_at = models.DateTimeField(null=True, blank=True)
    modified_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        managed = False
        db_table = 'customized_views'
        
    def __str__(self):
        return self.name
*****************************************************************
@action(detail=False, methods=['get'])
def explore(self, request):
    """
    FINAL VERSION - Using customized_views with url_id field!
    """
    
    from .models import CustomizedView
    from django.db import connection
    
    search = request.query_params.get('search', '')
    limit = int(request.query_params.get('limit', 100))
    
    with connection.cursor() as cursor:
        # Join customized_views with views, workbooks, and sites
        cursor.execute("""
            SELECT 
                cv.id,
                cv.name as customized_name,
                cv.url_id,
                v.name as view_name,
                v.repository_url as view_repo,
                v.index as view_index,
                w.name as workbook_name,
                w.repository_url as workbook_repo,
                s.name as site_name,
                s.url_namespace as site_url_namespace,
                cv.accessed_at,
                cv.public
            FROM customized_views cv
            JOIN views v ON cv.view_id = v.id
            JOIN workbooks w ON v.workbook_id = w.id
            JOIN sites s ON cv.site_id = s.id
            WHERE (cv.name ILIKE %s 
                OR v.name ILIKE %s 
                OR w.name ILIKE %s)
              AND cv.hidden = FALSE
            ORDER BY cv.accessed_at DESC NULLS LAST
            LIMIT %s
        """, [f'%{search}%', f'%{search}%', f'%{search}%', limit])
        
        rows = cursor.fetchall()
    
    results = []
    for r in rows:
        # Build URL using url_id from customized_views
        url_with_url_id = f"https://tableau.cib.echonet/#/site/ci5/views/{r[2]}"  # Using url_id
        
        # Also try with workbook_repo + view name
        view_name_clean = r[4].split('/')[-1] if r[4] else r[3]
        url_with_repo = f"https://tableau.cib.echonet/#/site/ci5/views/{r[7]}/{view_name_clean}?:iid={r[5]}"
        
        # Try with workbook + view name directly
        url_simple = f"https://tableau.cib.echonet/#/site/ci5/views/{r[7]}/{r[3]}?:iid={r[5]}"
        
        results.append({
            'id': r[0],
            'customized_name': r[1],
            'url_id': r[2],  # Special URL ID!
            'view_name': r[3],
            'view_repository_url': r[4],
            'view_index': r[5],
            'workbook_name': r[6],
            'workbook_repo_url': r[7],
            'site_name': r[8],
            'site_url_namespace': r[9],
            'last_accessed': r[10].isoformat() if r[10] else None,
            'is_public': r[11],
            'url_attempt_1_url_id': url_with_url_id,
            'url_attempt_2_repo': url_with_repo,
            'url_attempt_3_simple': url_simple,
        })
    
    return Response({
        'count': len(results),
        'message': 'Using customized_views with url_id field!',
        'note': 'These are user-accessed views, sorted by most recent',
        'search': search if search else 'all',
        'results': results
    })
