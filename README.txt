"use client";

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import authApi from "@/api";
import type { User } from "@/types";

const SSO_LOGIN_URL =
  "https://portal.qdi.dev.echonext/accounts/oidc/bnpp-oidc/login/";

export function useRequireAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const router = useRouter();

  useEffect(() => {
    let active = true;

    async function getUser() {
      try {
        const u = await authApi.getCurrentUser(); // GET /api/auth/users/me/

        if (active && u) {
          setUser(u);
        }
      } catch (err: any) {
        console.error("Auth error (me endpoint):", err);

        // AxiosError شکلش چیزی شبیه اینه: err.response.status
        const status = err?.response?.status;

        // هرچی جز 2xx باشه ما براش ارزش قائل نیستیم، می‌فرستیم SSO
        if (!status || status === 401 || status === 403) {
          // برای اطمینان کامل، به جای router.push از window.location استفاده می‌کنیم
          if (typeof window !== "undefined") {
            window.location.href = SSO_LOGIN_URL;
          } else {
            router.push(SSO_LOGIN_URL);
          }
        }
      } finally {
        if (active) setLoading(false);
      }
    }

    getUser();

    return () => {
      active = false;
    };
  }, [router]);

  return { user, loading };
}