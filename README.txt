import logging
from django.shortcuts import redirect
from django.conf import settings
from django.http import JsonResponse

logger = logging.getLogger(__name__)

class AuthValidationMiddleware:
    """
    Nuclear option: Validate EVERY request has valid auth or kick to SSO
    """
    def __init__(self, get_response):
        self.get_response = get_response
        
    def __call__(self, request):
        # Skip ONLY for these specific paths
        skip_paths = [
            '/admin/',
            '/api/health/',  # Keep this for healthcheck
            '/accounts/oidc/',  # SSO callback paths
            '/accounts/login/',
            '/static/',
            '/media/',
        ]
        
        # If path should skip validation, continue
        if any(request.path.startswith(path) for path in skip_paths):
            return self.get_response(request)
        
        # For API endpoints - return 401 JSON instead of redirect
        is_api = request.path.startswith('/api/')
        
        # Check if user is authenticated
        if not request.user.is_authenticated:
            logger.warning(f"User not authenticated for: {request.path}")
            
            # Kill any zombie session
            if request.session.session_key:
                logger.info(f"Flushing invalid session: {request.session.session_key}")
                request.session.flush()
            
            if is_api:
                return JsonResponse({
                    'authenticated': False,
                    'error': 'Authentication required'
                }, status=401)
            
            # Redirect to OIDC login for regular pages
            return redirect(f'{settings.PUBLIC_DOMAIN}/accounts/oidc/bnpp-oidc/login/')
        
        # User is authenticated, but validate session has required data
        required_session_keys = ['_auth_user_id', '_auth_user_backend']
        
        if not all(key in request.session for key in required_session_keys):
            logger.warning(f"Session missing required keys for user {request.user.id}")
            request.session.flush()
            
            if is_api:
                return JsonResponse({
                    'authenticated': False,
                    'error': 'Invalid session'
                }, status=401)
            
            return redirect(f'{settings.PUBLIC_DOMAIN}/accounts/oidc/bnpp-oidc/login/')
        
        logger.info(f"✓ Auth valid for {request.user.email} on {request.path}")
        
        # All good, continue
        response = self.get_response(request)
        return response
