@action(detail=False, methods=['get'])
def explore(self, request):
    """
    Fetch from public.views with proper URL building
    """
    
    from django.db import connection
    
    search = request.query_params.get('search', 'ClientMetric')
    
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT 
                v.id,
                v.name,
                v.repository_url,
                v.index,
                v.title,
                w.name as workbook_name,
                w.repository_url as workbook_repo_url,
                s.name as site_name,
                s.url_namespace as site_url_namespace
            FROM views v
            JOIN workbooks w ON v.workbook_id = w.id
            JOIN sites s ON v.site_id = s.id
            WHERE (v.name ILIKE %s OR w.name ILIKE %s OR v.title ILIKE %s)
              AND v.is_deleted = FALSE
              AND w.is_deleted = FALSE
            ORDER BY v.name
            LIMIT 50
        """, [f'%{search}%', f'%{search}%', f'%{search}%'])
        
        rows = cursor.fetchall()
    
    results = []
    for r in rows:
        # Extract view name from repository_url (last part after /)
        repo_url = r[2] if r[2] else ''
        view_url_part = repo_url.split('/')[-1] if '/' in repo_url else r[1]
        
        # Remove /sheets/ if present
        repo_url_clean = repo_url.replace('/sheets/', '/')
        
        # Build URL attempts
        url1 = f"https://tableau.cib.echonet/#/site/ci5/views/{r[6]}/{view_url_part}?:iid={r[3]}"
        url2 = f"https://tableau.cib.echonet/#/site/ci5/views/{r[6]}/{r[1]}?:iid={r[3]}"
        url3 = f"https://tableau.cib.echonet/#/site/ci5/views/{repo_url_clean}?:iid={r[3]}"
        
        results.append({
            'id': r[0],
            'view_name': r[1],
            'repository_url': r[2],
            'view_index': r[3],
            'view_title': r[4],
            'workbook_name': r[5],
            'workbook_repo_url': r[6],
            'site_name': r[7],
            'site_url_namespace': r[8],
            'url_attempt_1': url1,
            'url_attempt_2': url2,
            'url_attempt_3': url3,
        })
    
    return Response({
        'count': len(results),
        'search_term': search,
        'message': '3 URL attempts for each dashboard - test them!',
        'results': results
    })
