'use client';

import { Home } from '@/components/features/home/home';
import { useEffect, useState } from 'react';

export default function HomePage() {
  const [checking, setChecking] = useState(true);
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  useEffect(() => {
    const checkAuth = async () => {
      try {
        const response = await fetch('/api/auth/me/', { credentials: 'include' });
        if (response.ok) {
          // User is authenticated, show home
          setIsAuthenticated(true);
        } else {
          // Not authenticated, redirect to SSO
          window.location.href = `${process.env.NEXT_PUBLIC_API_URL || 'https://portal.qdi.dev.echonet'}/accounts/openid_connect/bnpp-oidc/login/?process=login`;
        }
      } catch (error) {
        // Error or not authenticated, redirect to SSO
        window.location.href = `${process.env.NEXT_PUBLIC_API_URL || 'https://portal.qdi.dev.echonet'}/accounts/openid_connect/bnpp-oidc/login/?process=login`;
      } finally {
        setChecking(false);
      }
    };

    checkAuth();
  }, []);

  if (checking) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-center">
          <div className="animate-spin rounded-full h-16 w-16 border-b-4 border-primary-600 mx-auto mb-4"></div>
          <p className="text-xl">Loading...</p>
        </div>
      </div>
    );
  }

  return isAuthenticated ? <Home /> : null;
}
