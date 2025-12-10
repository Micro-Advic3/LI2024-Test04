from datetime import datetime, timedelta

class CookieNukerMiddleware:
    """
    Force fresh SSO authentication by clearing old cookies
    """
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        path = request.path
        
        # Skip these paths completely - don't interfere with auth flow
        skip_paths = [
            '/api/health/',
            '/accounts/',
            '/admin/',
            '/static/',
            '/media/',
        ]
        
        for skip in skip_paths:
            if path.startswith(skip):
                return self.get_response(request)
        
        # If user is NOT authenticated, force SSO login
        if not request.user.is_authenticated:
            from django.shortcuts import redirect
            
            # Flush the session
            request.session.flush()
            
            # Create redirect response to SSO
            response = redirect('/accounts/oidc/bnpp-oidc/login/')
            
            # List of cookies to delete
            cookies_to_nuke = [
                'sessionid',
                'csrftoken',
                'messages',
                'SMSESSION',
                'SMIDENTITY',
                'SMAGENTNAME',
            ]
            
            # List of domains to try
            domains = [
                '.qdi.dev.echonet',
                'qdi.dev.echonet',
                'portal.qdi.dev.echonet',
                '.dev.echonet',
                'dev.echonet',
                None,  # No domain
            ]
            
            # Delete cookies with all domain variations
            for cookie in cookies_to_nuke:
                for domain in domains:
                    try:
                        if domain:
                            response.delete_cookie(cookie, domain=domain)
                        else:
                            response.delete_cookie(cookie)
                    except:
                        pass
            
            return response
        
        # User is authenticated, let them through
        return self.get_response(request)
