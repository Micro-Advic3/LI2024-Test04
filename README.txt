from django.shortcuts import redirect
import logging

logger = logging.getLogger(__name__)

class ForceReauthMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        path = request.path
        
        # Always skip these
        skip_paths = [
            '/accounts/',
            '/api/health/',
            '/admin/',
            '/static/',
            '/media/',
        ]
        
        for skip in skip_paths:
            if path.startswith(skip):
                return self.get_response(request)
        
        # Check if user is authenticated AND has valid portal session
        portal_session_verified = request.session.get('portal_auth_verified', False)
        
        # If user is NOT authenticated OR doesn't have portal session
        if not request.user.is_authenticated or not portal_session_verified:
            # Skip if this is the SSO callback
            if request.GET.get('code'):
                # Mark session as verified after successful SSO
                request.session['portal_auth_verified'] = True
                return self.get_response(request)
            
            # Force fresh SSO authentication
            logger.info(f"Forcing fresh SSO authentication from: {path}")
            request.session.flush()
            response = redirect('/accounts/oidc/bnpp-oidc/login/?prompt=login')
            
            # Clear ALL cookies to force fresh auth
            response.delete_cookie('sessionid', domain='.qdi.dev.echonet')
            response.delete_cookie('csrftoken', domain='.qdi.dev.echonet')
            response.delete_cookie('messages', domain='.qdi.dev.echonet')
            response.delete_cookie('SMSESSION', domain='.dev.echonet')  # SSO cookie
            response.delete_cookie('SMSESSION')  # Without domain too
            
            return response
        
        return self.get_response(request)
