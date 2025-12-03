class SocialAccountAdapter(DefaultSocialAccountAdapter):
    # ... your existing methods ...
    
    def get_app(self, request, provider, client_id=None):
        """
        Override to fix MultipleObjectsReturned bug
        There's only 1 OIDC app, so just return it
        """
        from allauth.socialaccount.models import SocialApp
        from django.contrib.sites.shortcuts import get_current_site
        
        try:
            # Get the current site
            site = get_current_site(request)
            
            # Get the OIDC app for this site
            return SocialApp.objects.get(
                provider='openid_connect',
                provider_id='bnpp-oidc',
                sites=site
            )
        except SocialApp.DoesNotExist:
            return None
        except SocialApp.MultipleObjectsReturned:
            # If somehow there are duplicates, just get the first one
            return SocialApp.objects.filter(
                provider='openid_connect',
                provider_id='bnpp-oidc',
                sites=site
            ).first()
