'use client';

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import { authApi } from "@/api";
import type {User} from '@/types/index';

export function useRequireAuth(){
  const [user, setUser] = useState<User|null>(null);
  const [loading, setLoading] = useState(true);
  const router = useRouter();

  useEffect(() => {
    let active = true;
    
    async function checkAndGetUser() {
      try {
        // STEP 1: Check if sessionid cookie exists
        const sessionCookie = document.cookie
          .split(';')
          .find(cookie => cookie.trim().startsWith('sessionid='));
        
        if (!sessionCookie) {
          console.log('[AUTH] No session cookie, redirecting to SSO...');
          window.location.href = `${window.location.origin}/accounts/oidc/bnpp-oidc/login/`;
          return;
        }
        
        // STEP 2: Validate session cookie format (Django session format)
        const sessionValue = sessionCookie.split('=')[1];
        
        // Django sessions are typically 32+ alphanumeric characters
        const isValidFormat = sessionValue && sessionValue.length >= 32 && /^[a-zA-Z0-9]+$/.test(sessionValue);
        
        if (!isValidFormat) {
          console.log('[AUTH] Invalid session format detected, clearing and redirecting...');
          
          // Clear ALL cookies
          document.cookie.split(';').forEach(cookie => {
            const name = cookie.split('=')[0].trim();
            document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; domain=.qdi.dev.echonet`;
            document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; domain=.dev.echonet`;
            document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;`;
          });
          
          // Redirect to SSO
          window.location.href = `${window.location.origin}/accounts/oidc/bnpp-oidc/login/`;
          return;
        }
        
        // STEP 3: Session looks valid, try to get user
        console.log('[AUTH] Valid session cookie found, getting user data...');
        const currentUser = await authApi.getCurrentUser();
        
        if (active && currentUser) {
          setUser(currentUser);
          console.log('[AUTH] User authenticated:', currentUser);
        } else {
          // Session exists but invalid, redirect
          window.location.href = `${window.location.origin}/accounts/oidc/bnpp-oidc/login/`;
        }
      } catch (error) {
        console.error('[AUTH] Error during auth:', error);
        
        if (!active) return;
        
        // Clear cookies and redirect on error
        document.cookie.split(';').forEach(cookie => {
          const name = cookie.split('=')[0].trim();
          document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; domain=.qdi.dev.echonet`;
          document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; domain=.dev.echonet`;
          document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;`;
        });
        
        window.location.href = `${window.location.origin}/accounts/oidc/bnpp-oidc/login/`;
      } finally {
        if (active) setLoading(false);
      }
    }
    
    checkAndGetUser();
    
    return () => {
      active = false;
    };
  }, [router]);

  return {user, loading};
}
