## Frontend
# Frontend Changes — OTP Email Verification

Backend now returns **no tokens** from `/auth/register`, and `/auth/login`
returns **403** for unverified accounts. The frontend needs: updated types,
updated service calls, a new OTP verify page, and updated signup/login flows
to route users through it.

---

## 1. MODIFIED — `types/auth.ts`

```typescript
import { User } from "./user";


export interface TokenResponse{
    access_token: string;
    refresh_token: string;
    token_type: string;
}

export interface AuthResponse {
    message: string;
    user: User;
    tokens: TokenResponse;
}

// NEW — register no longer returns tokens (account is unverified at this point)
export interface RegisterResponse {
    message: string;
    user: User;
}

// NEW
export interface SendOtpRequest {
    email: string;
    purpose?: string; // defaults to "email_verification" on backend
}

// NEW
export interface VerifyOtpRequest {
    email: string;
    otp: string;
    purpose?: string;
}
```

---

## 2. MODIFIED — `services/auth.ts`

```typescript
import api from "@/lib/api";
import { User } from "@/types/user";
import { AuthResponse, RegisterResponse, SendOtpRequest, VerifyOtpRequest } from "@/types/auth";

export async function login(
            email: string, 
            password: string
        ): Promise<AuthResponse> {
        
            const response = await api.post("/auth/login", {email,password});
            return response.data;
    }

export async function getCurrentUser(): Promise<User> {

        const response = await api.get("/auth/me");
        return response.data;
}

// CHANGED: return type is now RegisterResponse (message + user, no tokens)
export async function register(
    name: string,
    email: string,
    password: string
): Promise<RegisterResponse> {
    const response = await api.post("/auth/register", { email, password, name });
    return response.data;
}

export async function requestPasswordReset(email: string): Promise<void> {
    await api.post("/auth/forgot-password", { email });
}

export async function resetPassword(token: string, password: string): Promise<void> {
    await api.put("/auth/reset-password", { token, password });
}

export async function googleLogin(credential: string): Promise<AuthResponse> {
    const response = await api.post("/auth/google", { token: credential });
    return response.data;
}

export async function refreshSession(refreshToken: string): Promise<AuthResponse> {
    const response = await api.post("/auth/refresh", {
        refresh_token: refreshToken,
    });
    return response.data;
}

// NEW
export async function sendOtp(email: string, purpose = "email_verification"): Promise<void> {
    const payload: SendOtpRequest = { email, purpose };
    await api.post("/auth/send-otp", payload);
}

// NEW — this is the endpoint that actually returns tokens once the OTP is correct
export async function verifyOtp(
    email: string,
    otp: string,
    purpose = "email_verification"
): Promise<AuthResponse> {
    const payload: VerifyOtpRequest = { email, otp, purpose };
    const response = await api.post("/auth/verify-otp", payload);
    return response.data;
}
```

---

## 3. MODIFIED — `store/auth-store.ts`

Key changes:
- `signup` no longer marks the user authenticated (no tokens come back). It just
  creates the account and returns nothing useful to store — the signup **page**
  is responsible for redirecting to `/verify-otp`.
- New `verifyOtp` and `resendOtp` actions — `verifyOtp` is what actually stores
  tokens and flips `isAuthenticated`.

```typescript
import {
  getCurrentUser as getUser,
  googleLogin,
  login as loginUser,
  refreshSession,
  register as registerUser,
  sendOtp,
  verifyOtp as verifyOtpRequest,
} from "@/services/auth";
import { User } from "@/types/user";
import { create } from "zustand";
import {
  clearAuthTokens,
  getAccessToken,
  getRefreshToken,
  isDesktop,
  redirectToDesktopLogin,
  shouldRefreshDesktopAccessToken,
  storeAuthTokens,
} from "@/lib/desktop";

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  isInitialized: boolean;
  token: string | null;

  login: (email: string, password: string) => Promise<void>;
  signup: (name: string, email: string, password: string) => Promise<void>;
  verifyOtp: (email: string, otp: string) => Promise<void>;
  resendOtp: (email: string) => Promise<void>;
  loginWithGoogle: (credential: string) => Promise<void>;
  logout: () => void;
  restoreSession: () => Promise<void>;
  setUser: (user: User) => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  isLoading: false,
  isInitialized: false,
  token: typeof window !== "undefined" ? getAccessToken() : null,

  login: async (email, password) => {
    const data = await loginUser(email, password);
    storeAuthTokens(data.tokens);
    set({ user: data.user, isAuthenticated: true, token: data.tokens.access_token });
  },

  // CHANGED: no tokens come back from register anymore. Account exists but
  // is unverified — do NOT set isAuthenticated. The signup page redirects
  // to /verify-otp on success.
  signup: async (name, email, password) => {
    await registerUser(name, email, password);
  },

  // NEW
  verifyOtp: async (email, otp) => {
    const data = await verifyOtpRequest(email, otp);
    storeAuthTokens(data.tokens);
    set({ user: data.user, isAuthenticated: true, token: data.tokens.access_token });
  },

  // NEW
  resendOtp: async (email) => {
    await sendOtp(email);
  },

  loginWithGoogle: async (credential) => {
    const data = await googleLogin(credential);
    storeAuthTokens(data.tokens);
    set({ user: data.user, isAuthenticated: true, token: data.tokens.access_token });
  },

  logout: () => {
    clearAuthTokens();
    set({ user: null, isAuthenticated: false, token: null });
  },

  restoreSession: async () => {
    set({ isLoading: true });
    const finish = (state: Partial<AuthState>) => {
      set({
        isLoading: false,
        isInitialized: true,
        ...state,
      });
    };

    const accessToken = getAccessToken();
    const refreshToken = getRefreshToken();
    const restoreWithRefreshToken = async () => {
      if (!refreshToken) {
        return false;
      }

      const refreshed = await refreshSession(refreshToken);
      storeAuthTokens(refreshed.tokens);
      const data = await getUser();
      finish({
        user: data,
        isAuthenticated: true,
        token: refreshed.tokens.access_token,
      });
      return true;
    };

    try {
      if (isDesktop() && shouldRefreshDesktopAccessToken() && refreshToken) {
        try {
          if (await restoreWithRefreshToken()) {
            return;
          }
        } catch {
          // Fall back to the access token path below if refresh fails.
        }
      }

      if (accessToken) {
        try {
          const data = await getUser();
          finish({
            user: data,
            isAuthenticated: true,
            token: getAccessToken(),
          });
          return;
        } catch {
          if (!refreshToken) {
            throw new Error("Session expired");
          }
        }
      }

      if (refreshToken) {
        await restoreWithRefreshToken();
        return;
      }

      finish({
        user: null,
        isAuthenticated: false,
        token: null,
      });
    } catch {
      clearAuthTokens();
      if (isDesktop()) {
        redirectToDesktopLogin("session-expired");
      }
      finish({
        user: null,
        isAuthenticated: false,
        token: null,
      });
    }
  },

  setUser: (user) => set({ user }),
}));
```

---

## 4. NEW — `app/(auth)/verify-otp/page.tsx`

Reads `email` from the query string (passed in by signup/login redirects),
shows a 6-digit input, calls `verifyOtp`, and on success goes to
`/avatar-setup` (same place signup used to redirect to before this feature —
check your `avatar-setup` gate in `app/(app)/layout.tsx`, it's unchanged).

```tsx
"use client";

import { Suspense, useState, useRef, useEffect } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import Link from "next/link";
import { Loader2, MailCheck, ArrowLeft } from "lucide-react";
import { useAuthStore } from "@/store/auth-store";
import { isAxiosError } from "axios";

function VerifyOtpForm() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const email = searchParams.get("email") ?? "";

  const verifyOtp = useAuthStore((state) => state.verifyOtp);
  const resendOtp = useAuthStore((state) => state.resendOtp);

  const [digits, setDigits] = useState<string[]>(Array(6).fill(""));
  const [error, setError] = useState("");
  const [isVerifying, setIsVerifying] = useState(false);
  const [isResending, setIsResending] = useState(false);
  const [resent, setResent] = useState(false);
  const inputRefs = useRef<(HTMLInputElement | null)[]>([]);

  useEffect(() => {
    inputRefs.current[0]?.focus();
  }, []);

  const otp = digits.join("");

  function handleChange(index: number, value: string) {
    if (!/^\d*$/.test(value)) return;

    const next = [...digits];
    next[index] = value.slice(-1);
    setDigits(next);

    if (value && index < 5) {
      inputRefs.current[index + 1]?.focus();
    }
  }

  function handleKeyDown(index: number, e: React.KeyboardEvent<HTMLInputElement>) {
    if (e.key === "Backspace" && !digits[index] && index > 0) {
      inputRefs.current[index - 1]?.focus();
    }
  }

  function handlePaste(e: React.ClipboardEvent<HTMLInputElement>) {
    const pasted = e.clipboardData.getData("text").replace(/\D/g, "").slice(0, 6);
    if (!pasted) return;
    e.preventDefault();
    const next = Array(6).fill("");
    for (let i = 0; i < pasted.length; i++) next[i] = pasted[i];
    setDigits(next);
    inputRefs.current[Math.min(pasted.length, 5)]?.focus();
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (otp.length !== 6) {
      setError("Enter the full 6-digit code.");
      return;
    }

    setError("");
    setIsVerifying(true);
    try {
      await verifyOtp(email, otp);
      router.push("/avatar-setup");
    } catch (err: unknown) {
      if (isAxiosError(err) && err.response?.data?.detail) {
        setError(err.response.data.detail);
      } else {
        setError("Invalid or expired code. Please try again.");
      }
      setDigits(Array(6).fill(""));
      inputRefs.current[0]?.focus();
    } finally {
      setIsVerifying(false);
    }
  }

  async function handleResend() {
    setError("");
    setIsResending(true);
    setResent(false);
    try {
      await resendOtp(email);
      setResent(true);
      setDigits(Array(6).fill(""));
      inputRefs.current[0]?.focus();
    } catch {
      setError("Could not resend the code. Please try again shortly.");
    } finally {
      setIsResending(false);
    }
  }

  if (!email) {
    return (
      <div className="w-full max-w-[420px] p-4">
        <div className="card rounded-3xl bg-card/60 p-8 shadow-xl backdrop-blur-xl text-center">
          <p className="mb-4 text-sm text-muted-foreground">
            We couldn&apos;t find an email to verify. Please sign up or log in again.
          </p>
          <Link href="/signup" className="text-sm font-semibold text-primary">
            Back to sign up
          </Link>
        </div>
      </div>
    );
  }

  return (
    <div className="w-full max-w-[420px] p-4">
      <div className="card relative overflow-hidden rounded-[1.75rem] border border-border/60 p-6 shadow-[0_16px_50px_rgba(0,0,0,0.10)] sm:rounded-[2rem] sm:p-8 sm:shadow-[0_24px_80px_rgba(0,0,0,0.14)] sm:backdrop-blur-2xl">
        <div className="absolute inset-x-0 top-0 h-1 bg-gradient-to-r from-primary via-primary/50 to-transparent" />

        <Link
          href="/login"
          className="mb-6 inline-flex items-center gap-1.5 text-sm text-muted-foreground"
        >
          <ArrowLeft size={14} />
          Back to login
        </Link>

        <div className="mb-7 text-center sm:mb-8">
          <div className="mx-auto mb-4 flex size-12 items-center justify-center rounded-2xl bg-primary/10 text-primary">
            <MailCheck className="size-6" />
          </div>
          <p className="text-[10px] font-semibold uppercase tracking-[0.3em] text-muted-foreground">
            Verify your email
          </p>
          <h1 className="mt-3 text-2xl font-semibold tracking-tight sm:text-3xl">
            Enter the code
          </h1>
          <p className="mx-auto mt-3 max-w-sm text-sm leading-6 text-muted-foreground">
            We sent a 6-digit code to <span className="font-medium text-foreground">{email}</span>.
            It expires in a few minutes.
          </p>
        </div>

        <form onSubmit={handleSubmit} className="space-y-5">
          <div className="flex justify-between gap-2">
            {digits.map((digit, index) => (
              <input
                key={index}
                ref={(el) => {
                  inputRefs.current[index] = el;
                }}
                type="text"
                inputMode="numeric"
                maxLength={1}
                value={digit}
                onChange={(e) => handleChange(index, e.target.value)}
                onKeyDown={(e) => handleKeyDown(index, e)}
                onPaste={handlePaste}
                className="h-14 w-full rounded-2xl border border-border/60 bg-background/55 text-center text-xl font-semibold outline-none transition-all duration-200 focus:border-primary focus:bg-background"
              />
            ))}
          </div>

          {error && (
            <div className="rounded-2xl border border-red-900/30 bg-red-950/20 p-3 text-sm leading-6 text-red-300 animate-in fade-in duration-200">
              {error}
            </div>
          )}

          {resent && !error && (
            <div className="rounded-2xl border border-emerald-900/30 bg-emerald-950/20 p-3 text-sm leading-6 text-emerald-300 animate-in fade-in duration-200">
              A new code has been sent.
            </div>
          )}

          <button
            type="submit"
            disabled={isVerifying || otp.length !== 6}
            className="flex h-12 w-full items-center justify-center gap-2 rounded-2xl bg-primary text-base font-medium text-primary-foreground shadow-lg shadow-primary/10 transition-transform duration-200 active:scale-[0.99] disabled:opacity-50"
          >
            {isVerifying && <Loader2 className="size-4 animate-spin" />}
            {isVerifying ? "Verifying…" : "Verify email"}
          </button>
        </form>

        <div className="mt-6 text-center text-sm text-muted-foreground">
          Didn&apos;t get a code?{" "}
          <button
            type="button"
            onClick={handleResend}
            disabled={isResending}
            className="font-semibold text-primary transition-colors hover:text-primary/80 disabled:opacity-50"
          >
            {isResending ? "Sending…" : "Resend code"}
          </button>
        </div>
      </div>
    </div>
  );
}

export default function VerifyOtpPage() {
  return (
    <Suspense
      fallback={
        <div className="w-full max-w-[420px] p-4">
          <div className="card rounded-3xl bg-card/60 p-8 shadow-xl backdrop-blur-xl">
            <div className="mx-auto size-5 animate-spin rounded-full border-2 border-primary border-r-transparent" />
          </div>
        </div>
      }
    >
      <VerifyOtpForm />
    </Suspense>
  );
}
```

This page lives inside `app/(auth)/`, so it automatically gets `AuthShell` /
`AuthSlideShell` from `app/(auth)/layout.tsx` — no extra layout work needed.

---

## 5. MODIFIED — `app/(auth)/signup/page.tsx`

Only `handleSubmit` changes — redirect to `/verify-otp?email=...` instead of
`/avatar-setup`, since there's no session yet.

```tsx
"use client";

import React, { useState } from "react";
import { useRouter } from "next/navigation";
import { Eye, EyeOff } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import Link from "next/link";

import { useAuthStore } from "@/store/auth-store";
import { GoogleSignInButton } from "@/components/auth/GoogleSignInButton";

export default function Register() {
  const router = useRouter();
  const signup = useAuthStore((state) => state.signup);

  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [error, setError] = useState("");
  const [isLoading, setIsLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setError("");
    setIsLoading(true);
    try {
      await signup(name, email, password);
      // CHANGED: no session yet — the account needs OTP verification first.
      router.push(`/verify-otp?email=${encodeURIComponent(email)}`);
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : "Something went wrong.");
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <div className="relative isolate w-full max-w-[420px] animate-in fade-in slide-in-from-bottom-4 duration-500 lg:max-w-[400px] rounded-[2rem] overflow-hidden bg-(--evven-card-background) border border-border/60 shadow-[0_16px_50px_rgba(0,0,0,0.10)] sm:shadow-[0_24px_80px_rgba(0,0,0,0.14)] sm:backdrop-blur-2xl">
      <div className="hidden -inset-6 -z-10 rounded-4xl bg-gradient-to-br from-primary/10 via-primary/5 to-transparent opacity-60 blur-2xl md:absolute md:block" />

      <div className="relative overflow-hidden rounded-[1.75rem] border border-border/60 bg-card/90 p-6 shadow-[0_16px_50px_rgba(0,0,0,0.10)] sm:rounded-[2rem] sm:p-8 sm:shadow-[0_24px_80px_rgba(0,0,0,0.14)] sm:backdrop-blur-2xl">
        <div className="absolute inset-x-0 top-0 h-1 bg-gradient-to-r from-primary via-primary/50 to-transparent" />

        <div className="mb-7 text-center sm:mb-8">
          <p className="text-[10px] font-semibold uppercase tracking-[0.3em] text-muted-foreground">
            Sign up
          </p>
          <h1 className="mt-3 text-[1.9rem] font-semibold tracking-tight sm:text-[2.6rem]">
            Create an account.
          </h1>
          <p className="mx-auto mt-3 max-w-sm text-sm leading-6 text-muted-foreground">
            Join us and get started today.
          </p>
        </div>

        <form onSubmit={handleSubmit} className="space-y-4 sm:space-y-5">
          <div className="space-y-2">
            <Label htmlFor="name" className="text-sm font-medium">Full Name</Label>
            <Input
              id="name"
              type="text"
              placeholder="Your name"
              value={name}
              autoComplete="off"
              onChange={(e) => setName(e.target.value)}
              required
              className="h-12 rounded-2xl border-border/60 bg-background/55 transition-all duration-200 focus:border-primary focus:bg-background"
            />
          </div>

          <div className="space-y-2">
            <Label htmlFor="email" className="text-sm font-medium">Email</Label>
            <Input
              id="email"
              type="email"
              placeholder="you@example.com"
              value={email}
              autoComplete="off"
              onChange={(e) => setEmail(e.target.value)}
              required
              className="h-12 rounded-2xl border-border/60 bg-background/55 transition-all duration-200 focus:border-primary focus:bg-background"
            />
          </div>

          <div className="space-y-2">
            <Label htmlFor="password" className="text-sm font-medium">Password</Label>
            <div className="relative">
              <Input
                id="password"
                type={showPassword ? "text" : "password"}
                placeholder="••••••••"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                required
                className="h-12 rounded-2xl border-border/60 bg-background/55 pr-10 transition-all duration-200 focus:border-primary focus:bg-background"
              />
              <button
                type="button"
                onClick={() => setShowPassword((v) => !v)}
                aria-label={showPassword ? "Hide password" : "Show password"}
                className="absolute right-3 top-1/2 -translate-y-1/2 text-muted-foreground transition-colors hover:text-foreground"
              >
                {showPassword ? <EyeOff className="size-5" /> : <Eye className="size-5" />}
              </button>
            </div>
          </div>

          {error && (
            <div className="rounded-2xl border border-red-900/30 bg-red-950/20 p-3 text-sm leading-6 text-red-300 animate-in fade-in duration-200">
              {error}
            </div>
          )}

          <Button
            type="submit"
            className="h-12 w-full rounded-2xl text-base font-medium shadow-lg shadow-primary/10 transition-transform duration-200 active:scale-[0.99]"
            size="lg"
            disabled={isLoading}
          >
            {isLoading ? (
              <span className="flex items-center gap-2">
                <span className="inline-block size-4 animate-spin rounded-full border-2 border-primary-foreground border-r-transparent" />
                Creating account…
              </span>
            ) : (
              "Sign up"
            )}
          </Button>
        </form>

        <div className="relative my-6">
          <div className="absolute inset-0 flex items-center">
            <div className="w-full border-t border-border/60" />
          </div>
          <div className="relative flex justify-center text-xs uppercase">
            <span
              className="card px-4 text-muted-foreground border-none shadow-none"
              style={{ border: "none", boxShadow: "none" }}
            >
              Or continue with
            </span>
          </div>
        </div>

        <GoogleSignInButton />

        <div className="mt-7 text-center text-sm text-muted-foreground sm:mt-8">
          Already have an account?{" "}
          <Link href="/login" className="font-semibold text-primary transition-colors hover:text-primary/80">
            Log in
          </Link>
        </div>
      </div>
    </div>
  );
}
```

---

## 6. MODIFIED — `app/(auth)/login/page.tsx`

Only `handleSubmit` changes — detect the 403 "unverified" case and redirect to
`/verify-otp` with the email pre-filled, instead of just showing a red error
box the user can't act on.

```tsx
"use client";

import React, { useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import { Eye, EyeOff } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import Link from "next/link";
import { isAxiosError } from "axios";

import { useAuthStore } from "@/store/auth-store";
import { GoogleSignInButton } from "@/components/auth/GoogleSignInButton";

export default function Login() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const login = useAuthStore((state) => state.login);
  const reason = searchParams.get("reason");
  const isSessionExpired = reason === "session-expired";

  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [error, setError] = useState("");
  const [isLoading, setIsLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setError("");
    setIsLoading(true);
    try {
      await login(email, password);
      router.push("/dashboard");
    } catch (err: unknown) {
      // CHANGED: an unverified account gets 403'd by the backend — route
      // the user to the OTP screen instead of leaving them stuck on an error.
      if (isAxiosError(err) && err.response?.status === 403) {
        router.push(`/verify-otp?email=${encodeURIComponent(email)}`);
        return;
      }
      setError(err instanceof Error ? err.message : "Invalid email or password.");
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <div className="relative isolate w-full max-w-[420px] animate-in fade-in slide-in-from-bottom-4 duration-500 lg:max-w-[400px]">
      <div className="hidden -inset-6 -z-10 rounded-4xl bg-gradient-to-br from-primary/10 via-primary/5 to-transparent opacity-60 blur-2xl md:absolute md:block" />

      <div className="card relative overflow-hidden rounded-[1.75rem] border border-border/60 p-6 shadow-[0_16px_50px_rgba(0,0,0,0.10)] sm:rounded-[2rem] sm:p-8 sm:shadow-[0_24px_80px_rgba(0,0,0,0.14)] sm:backdrop-blur-2xl">
        <div className="absolute inset-x-0 top-0 h-1 bg-gradient-to-r from-primary via-primary/50 to-transparent" />

        <div className="mb-7 text-center sm:mb-8">
          {isSessionExpired && (
            <div className="mb-4 rounded-2xl border border-amber-500/30 bg-amber-500/10 px-4 py-3 text-left text-sm leading-6 text-amber-200">
              Your desktop session expired or the token was unavailable. Log in again to keep Evven open.
            </div>
          )}
          <p className="text-[10px] font-semibold uppercase tracking-[0.3em] text-muted-foreground">
            Log in
          </p>
          <h1 className="mt-3 text-[1.9rem] font-semibold tracking-tight sm:text-[2.6rem]">
            Welcome back.
          </h1>
          <p className="mx-auto mt-3 max-w-sm text-sm leading-6 text-muted-foreground">
            {isSessionExpired
              ? "We saved your place. Just log in again to continue."
              : "Enter your credentials to continue"}
          </p>
        </div>

        <form onSubmit={handleSubmit} className="space-y-4 sm:space-y-5">
          <div className="space-y-2">
            <Label htmlFor="email" className="text-sm font-medium">Email</Label>
            <Input
              id="email"
              type="email"
              placeholder="you@example.com"
              value={email}
              autoComplete="off"
              onChange={(e) => setEmail(e.target.value)}
              required
              className="h-12 rounded-2xl border-border/60 bg-background/55 transition-all duration-200 focus:border-primary focus:bg-background"
            />
          </div>

          <div className="space-y-2">
            <div className="flex items-center justify-between">
              <Label htmlFor="password" className="text-sm font-medium">Password</Label>
              <Link href="/forgot-password" className="text-xs font-medium text-primary hover:underline">
                Forgot password?
              </Link>
            </div>
            <div className="relative">
              <Input
                id="password"
                type={showPassword ? "text" : "password"}
                placeholder="••••••••"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                required
                className="h-12 rounded-2xl border-border/60 bg-background/55 pr-10 transition-all duration-200 focus:border-primary focus:bg-background"
              />
              <button
                type="button"
                onClick={() => setShowPassword((v) => !v)}
                aria-label={showPassword ? "Hide password" : "Show password"}
                className="absolute right-3 top-1/2 -translate-y-1/2 text-muted-foreground transition-colors hover:text-foreground"
              >
                {showPassword ? <EyeOff className="size-5" /> : <Eye className="size-5" />}
              </button>
            </div>
          </div>

          {error && (
            <div className="rounded-2xl border border-red-900/30 bg-red-950/20 p-3 text-sm leading-6 text-red-300 animate-in fade-in duration-200">
              {error}
            </div>
          )}

          <Button
            type="submit"
            className="h-12 w-full rounded-2xl text-base font-medium shadow-lg shadow-primary/10 transition-transform duration-200 active:scale-[0.99]"
            size="lg"
            disabled={isLoading}
          >
            {isLoading ? (
              <span className="flex items-center gap-2">
                <span className="inline-block size-4 animate-spin rounded-full border-2 border-primary-foreground border-r-transparent" />
                Signing in…
              </span>
            ) : (
              "Log in"
            )}
          </Button>
        </form>

        <div className="relative my-6">
          <div className="absolute inset-0 flex items-center">
            <div className="w-full border-t border-border/60" />
          </div>
          <div className="relative flex justify-center text-xs uppercase">
            <span
              className="inline-flex px-4 text-muted-foreground bg-(--evven-card-background)"
              style={{ border: "none", boxShadow: "none" }}
            >
              Or continue with
            </span>
          </div>
        </div>

        <GoogleSignInButton />

        <div className="mt-7 text-center text-sm text-muted-foreground sm:mt-8">
          Don&apos;t have an account?{" "}
          <Link href="/signup" className="font-semibold text-primary transition-colors hover:text-primary/80">
            Sign up
          </Link>
        </div>
      </div>
    </div>
  );
}
```

---

## 7. Google Sign-In — no changes needed

`components/auth/GoogleSignInButton.tsx` and `loginWithGoogle` in the store are
untouched. Since the backend now sets `is_verified=True` on both new Google
signups and linked accounts, Google users always get tokens back immediately —
they never see the OTP screen. That's intentional.

---

## Summary of new/changed files

| File | Change |
|---|---|
| `types/auth.ts` | `RegisterResponse` (no tokens), `SendOtpRequest`, `VerifyOtpRequest` |
| `services/auth.ts` | `register()` return type changed; added `sendOtp`, `verifyOtp` |
| `store/auth-store.ts` | `signup()` no longer authenticates; added `verifyOtp`, `resendOtp` |
| `app/(auth)/verify-otp/page.tsx` | **NEW** — 6-digit code entry screen |
| `app/(auth)/signup/page.tsx` | Redirects to `/verify-otp?email=...` instead of `/avatar-setup` |
| `app/(auth)/login/page.tsx` | On 403, redirects to `/verify-otp?email=...` instead of showing a dead-end error |
| `components/auth/GoogleSignInButton.tsx` | No change |

One thing to double check on your end: `middleware.ts` / route guards, if you
have any outside of `app/(app)/layout.tsx`'s client-side redirect — make sure
nothing tries to route an authenticated-but-unverified user anywhere except
`/verify-otp`, since such a state shouldn't really exist now (no tokens issued
pre-verification) but it's worth confirming there's no code path assuming
"has a user object" implies "is fully onboarded."


## Backend
# OTP Email Verification + Google Account Linking — Full Change Set

This document lists every file that needs to change: **NEW** files to create, and
**MODIFIED** files with their full updated content. Apply in the order listed —
later files import from earlier ones.

Also included at the end: the Alembic migration you'll need (new column +
new table), and a checklist of env vars to add.

---

## 1. NEW — `models/otp_verification.py`

```python
import uuid

from sqlalchemy import TIMESTAMP, Boolean, Column, ForeignKey, Integer, String
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.sql import func

from core.database import Base


class OtpVerification(Base):
    __tablename__ = "otp_verifications"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    otp_hash = Column(String, nullable=False)
    purpose = Column(String, nullable=False, default="email_verification")
    # "email_verification" | "google_link"
    expire_at = Column(TIMESTAMP(timezone=True), nullable=False)
    attempts = Column(Integer, nullable=False, default=0)
    used = Column(Boolean, default=False, nullable=False)
    created_at = Column(TIMESTAMP(timezone=True), server_default=func.now())
```

---

## 2. NEW — `repository/otp_repository.py`

```python
from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from models.otp_verification import OtpVerification


class OtpRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create_otp(
        self, user_id: UUID, otp_hash: str, purpose: str, expire_at: datetime
    ) -> OtpVerification:
        record = OtpVerification(
            user_id=user_id, otp_hash=otp_hash, purpose=purpose, expire_at=expire_at
        )
        self.session.add(record)
        await self.session.commit()
        await self.session.refresh(record)
        return record

    async def get_latest_active_otp(
        self, user_id: UUID, purpose: str
    ) -> OtpVerification | None:
        result = await self.session.execute(
            select(OtpVerification)
            .where(
                OtpVerification.user_id == user_id,
                OtpVerification.purpose == purpose,
                OtpVerification.used.is_(False),
                OtpVerification.expire_at > datetime.now(timezone.utc),
            )
            .order_by(OtpVerification.created_at.desc())
        )
        return result.scalars().first()

    async def increment_attempts(self, record: OtpVerification) -> None:
        record.attempts += 1
        await self.session.commit()

    async def mark_used(self, record: OtpVerification) -> None:
        record.used = True
        await self.session.commit()

    async def invalidate_existing(self, user_id: UUID, purpose: str) -> None:
        result = await self.session.execute(
            select(OtpVerification).where(
                OtpVerification.user_id == user_id,
                OtpVerification.purpose == purpose,
                OtpVerification.used.is_(False),
            )
        )
        for record in result.scalars().all():
            record.used = True
        await self.session.commit()
```

---

## 3. NEW — `services/otp_service.py`

```python
import hashlib
import random
from datetime import datetime, timedelta, timezone
from uuid import UUID

import resend
from fastapi import HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from core.config import OTP_EXPIRE_MINUTES, OTP_MAX_ATTEMPTS, RESEND_API_KEY, RESEND_FROM
from repository.otp_repository import OtpRepository
from repository.user_repository import UserRepository

resend.api_key = RESEND_API_KEY


def _generate_otp() -> str:
    return f"{random.randint(0, 999999):06d}"


def _hash_otp(otp: str) -> str:
    return hashlib.sha256(otp.encode()).hexdigest()


async def send_otp(user_id: UUID, email: str, purpose: str, db: AsyncSession) -> dict:
    otp_repo = OtpRepository(db)

    await otp_repo.invalidate_existing(user_id, purpose)

    raw_otp = _generate_otp()
    otp_hash = _hash_otp(raw_otp)
    expire_at = datetime.now(timezone.utc) + timedelta(minutes=OTP_EXPIRE_MINUTES)

    await otp_repo.create_otp(user_id, otp_hash, purpose, expire_at)

    try:
        resend.Emails.send(
            {
                "from": f"Evven <{RESEND_FROM}>",
                "to": [email],
                "subject": "Your verification code",
                "html": f"<p>Your verification code is <b>{raw_otp}</b>. "
                f"It expires in {OTP_EXPIRE_MINUTES} minutes.</p>",
            }
        )
    except Exception as e:
        print(f"[send_otp] failed for {email}: {e}")
        raise HTTPException(status_code=500, detail="Failed to send OTP email")

    return {"message": "OTP sent successfully"}


async def verify_otp(user_id: UUID, otp: str, purpose: str, db: AsyncSession) -> bool:
    otp_repo = OtpRepository(db)
    user_repo = UserRepository(db)

    record = await otp_repo.get_latest_active_otp(user_id, purpose)
    if not record:
        raise HTTPException(
            status_code=400, detail="No active OTP found, please request a new one"
        )

    if record.attempts >= OTP_MAX_ATTEMPTS:
        raise HTTPException(status_code=429, detail="Too many attempts, request a new OTP")

    if record.otp_hash != _hash_otp(otp):
        await otp_repo.increment_attempts(record)
        raise HTTPException(status_code=400, detail="Invalid OTP")

    await otp_repo.mark_used(record)

    if purpose == "email_verification":
        user = await user_repo.get_user_by_id(user_id)
        user.is_verified = True
        await user_repo.update_user(user)

    return True
```

---

## 4. MODIFIED — `models/user.py`

Only one line added (`is_verified`); rest is unchanged.

```python
import uuid
from enum import Enum

from sqlalchemy import TIMESTAMP, Boolean, Column, String  # type: ignore
from sqlalchemy import Enum as SQLEnum  # type: ignore
from sqlalchemy.dialects.postgresql import UUID  # type: ignore
from sqlalchemy.orm import relationship  # type: ignore
from sqlalchemy.sql import func  # type: ignore

from core.database import Base


class AuthProvider(Enum):
    LOCAL = "local"
    GOOGLE = "google"


class User(Base):
    __tablename__ = "users"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_code = Column(String, unique=True, nullable=False, index=True)
    name = Column(String, nullable=False)
    email = Column(String, unique=True, nullable=False)
    password_hash = Column(String, nullable=True)
    google_id = Column(String, unique=True, nullable=True)
    auth_provider = Column(
        SQLEnum(AuthProvider), nullable=False, default=AuthProvider.LOCAL
    )
    profile_picture = Column(String, nullable=True)
    is_active = Column(Boolean, nullable=False, default=True)
    is_verified = Column(Boolean, nullable=False, default=False)  # NEW
    created_at = Column(
        TIMESTAMP(timezone=True), server_default=func.now(), nullable=False
    )
    updated_at = Column(
        TIMESTAMP(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )

    # relationships

    personal_expenses = relationship("PersonalExpense", back_populates="user")
    groups = relationship(
        "Group", foreign_keys="Group.created_by", back_populates="creator"
    )
    group_memberships = relationship("GroupMember", back_populates="user")
    expenses_paid = relationship(
        "GroupExpense", foreign_keys="GroupExpense.paid_by", back_populates="payer"
    )
    expense_splits = relationship("ExpenseSplit", back_populates="user")
    payments_made = relationship(
        "Settlement", foreign_keys="Settlement.payer_id", back_populates="payer"
    )
    payments_received = relationship(
        "Settlement", foreign_keys="Settlement.receiver_id", back_populates="receiver"
    )

    # # relationship b/w friend and user

    friends = relationship(
        "Friend",
        back_populates="owner",
        cascade="all, delete-orphan",
    )
```

---

## 5. MODIFIED — `models/__init__.py`

```python
from models.expense_splits import ExpenseSplit as ExpenseSplit
from models.friend import Friend as Friend
from models.group_expenses import GroupExpense as GroupExpense
from models.group_members import GroupMember as GroupMember
from models.groups import Group as Group
from models.otp_verification import OtpVerification as OtpVerification  # NEW
from models.password_reset_token import PasswordResetToken as PasswordResetToken
from models.personal_expenses import PersonalExpense as PersonalExpense
from models.settlements import Settlement as Settlement
from models.user import User as User
```

---

## 6. MODIFIED — `core/config.py`

```python
import os

from dotenv import load_dotenv  # type: ignore

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")
SECRET_KEY = os.getenv("SECRET_KEY")
ALGORITHM = os.getenv("ALGORITHM")
ACCESS_TOKEN_EXPIRE_MINUTES = int(os.getenv("ACCESS_TOKEN_EXPIRE_MINUTES"))
REFRESH_TOKEN_EXPIRE_DAYS = int(os.getenv("REFRESH_TOKEN_EXPIRE_DAYS"))

RESET_TOKEN_EXPIRE_MINUTES = int(os.getenv("RESET_TOKEN_EXPIRE_MINUTES"))

# OTP verification
OTP_EXPIRE_MINUTES = int(os.getenv("OTP_EXPIRE_MINUTES", "10"))  # NEW
OTP_MAX_ATTEMPTS = int(os.getenv("OTP_MAX_ATTEMPTS", "5"))  # NEW

# Resend email
RESEND_API_KEY = os.getenv("RESEND_API_KEY")
RESEND_FROM = os.getenv("RESEND_FROM")

# celery
# CELERY_BROKER_URL = os.getenv("CELERY_BROKER_URL")
# CELERY_RESULT_BACKEND = os.getenv("CELERY_RESULT_BACKEND")

# backed frontend url
BACKEND_URL = os.getenv("BACKEND_URL")

# GOOGLE client id
GOOGLE_CLIENT_ID = os.getenv("GOOGLE_CLIENT_ID")
```

---

## 7. MODIFIED — `schemas/auth.py`

```python
from pydantic import BaseModel, EmailStr

from schemas.user import TokenResponse, UserResponse


class RegisterResponse(BaseModel):
    message: str
    user: UserResponse
    # tokens removed on purpose — issued only after OTP verification


class LoginResponse(BaseModel):
    message: str
    user: UserResponse
    tokens: TokenResponse


class RefreshTokenRequest(BaseModel):
    refresh_token: str


class ForgotPasswordRequest(BaseModel):
    email: EmailStr


class ResetPasswordRequest(BaseModel):
    token: str
    password: str


class SendOtpRequest(BaseModel):  # NEW
    email: EmailStr
    purpose: str = "email_verification"


class VerifyOtpRequest(BaseModel):  # NEW
    email: EmailStr
    otp: str
    purpose: str = "email_verification"
```

---

## 8. MODIFIED — `services/auth_service.py`

Changes: `register_user` no longer issues tokens and handles unverified retries;
`login_user` blocks unverified users; `google_login` links to an existing local
account instead of rejecting.

```python
from datetime import datetime, timedelta, timezone

from fastapi import HTTPException, status  # type: ignore
from google.auth.transport import requests as google_requests
from google.oauth2 import id_token as google_id_token
from jose import JWTError, jwt  # type: ignore
from passlib.context import CryptContext  # type: ignore
from sqlalchemy.ext.asyncio import AsyncSession  # type: ignore

from core.config import (
    ACCESS_TOKEN_EXPIRE_MINUTES,
    ALGORITHM,
    GOOGLE_CLIENT_ID,
    REFRESH_TOKEN_EXPIRE_DAYS,
    SECRET_KEY,
)
from models.user import AuthProvider, User
from repository.user_repository import UserRepository
from schemas.auth import LoginResponse, RegisterResponse
from schemas.user import GoogleAuthRequest, TokenResponse, UserCreate, UserLogin
from utils.user_utils import generate_user_code

pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(
            minutes=ACCESS_TOKEN_EXPIRE_MINUTES
        )
    to_encode.update(
        {
            "exp": expire,
            "type": "access",
        }
    )
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt


def create_refresh_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update(
        {
            "exp": expire,
            "type": "refresh",
        }
    )
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt


def decode_token(
    token: str,
    expected_type: str | None = None,
) -> dict | None:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

        if expected_type and payload.get("type") != expected_type:
            return None

        return payload

    except JWTError:
        return None


async def register_user(user_data: UserCreate, db: AsyncSession) -> RegisterResponse:
    repo = UserRepository(db)

    existing_user = await repo.get_user_by_email(user_data.email)
    if existing_user:
        if existing_user.is_verified:
            raise HTTPException(status_code=400, detail="Email already registered")

        if existing_user.auth_provider != AuthProvider.LOCAL:
            raise HTTPException(status_code=400, detail="Email already registered")

        # Unverified local account retrying registration — refresh their
        # details and resend the OTP instead of creating a duplicate row.
        existing_user.name = user_data.name
        existing_user.password_hash = hash_password(user_data.password)
        updated_user = await repo.update_user(existing_user)

        from services.otp_service import send_otp  # local import avoids circular import

        await send_otp(updated_user.id, updated_user.email, "email_verification", db)

        return RegisterResponse(
            message="Account already exists but is unverified. A new OTP has been sent.",
            user=updated_user,
        )

    while True:
        code = generate_user_code()

        existing = await repo.get_user_by_user_code(code)

        if not existing:
            break

    hashed_password = hash_password(user_data.password)

    new_user = User(
        user_code=code,
        name=user_data.name,
        email=user_data.email,
        password_hash=hashed_password,
        auth_provider=AuthProvider.LOCAL,
        is_verified=False,
    )

    created_user = await repo.create_user(new_user)

    from services.otp_service import send_otp

    await send_otp(created_user.id, created_user.email, "email_verification", db)

    return RegisterResponse(
        message="Registered successfully. Please verify the OTP sent to your email.",
        user=created_user,
    )


async def login_user(login_data: UserLogin, db: AsyncSession) -> LoginResponse:
    repo = UserRepository(db)

    user = await repo.get_user_by_email(login_data.email)
    if not user or not verify_password(login_data.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Invalid email or password")

    if not user.is_verified:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Email not verified. Please verify the OTP sent to your email.",
        )

    payload = {"sub": str(user.id)}

    access_token = create_access_token(payload)
    refresh_token = create_refresh_token(payload)

    return LoginResponse(
        message="Login successful",
        user=user,
        tokens=TokenResponse(
            access_token=access_token,
            refresh_token=refresh_token,
            token_type="bearer",
        ),
    )


async def google_login(auth_data: GoogleAuthRequest, db: AsyncSession) -> LoginResponse:

    repo = UserRepository(db)
    try:
        idinfo = google_id_token.verify_oauth2_token(
            auth_data.token, google_requests.Request(), GOOGLE_CLIENT_ID
        )
    except ValueError:
        raise HTTPException(status_code=401, detail="Invalid Google token")

    google_id = idinfo["sub"]
    email = idinfo.get("email")
    name = idinfo.get("name") or (email.split("@")[0] if email else "User")
    picture = idinfo.get("picture")

    if not email:
        raise HTTPException(status_code=400, detail="Google account has no email")

    user = await repo.get_user_by_google_id(google_id)

    if not user:
        existing_email_user = await repo.get_user_by_email(email)

        if existing_email_user:
            google_email_verified = idinfo.get("email_verified", False)

            if not google_email_verified:
                raise HTTPException(
                    status_code=400,
                    detail="Google account email is not verified, cannot link accounts",
                )

            # Link Google to the existing local account. Google itself
            # vouches for email ownership here (email_verified claim), so
            # this is safe even if the account's own is_verified is False.
            existing_email_user.google_id = google_id
            existing_email_user.is_verified = True
            if not existing_email_user.profile_picture:
                existing_email_user.profile_picture = picture
            user = await repo.update_user(existing_email_user)
        else:
            while True:
                code = generate_user_code()
                existing = await repo.get_user_by_user_code(code)
                if not existing:
                    break

            new_user = User(
                user_code=code,
                name=name,
                email=email,
                google_id=google_id,
                auth_provider=AuthProvider.GOOGLE,
                profile_picture=picture,
                is_verified=True,  # Google already verified this email
            )
            user = await repo.create_user(new_user)

    payload = {"sub": str(user.id)}
    access_token = create_access_token(payload)
    refresh_token = create_refresh_token(payload)

    return LoginResponse(
        message="Google login successful",
        user=user,
        tokens=TokenResponse(
            access_token=access_token,
            refresh_token=refresh_token,
            token_type="bearer",
        ),
    )
```

---

## 9. MODIFIED — `routes/auth.py`

```python
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.responses import FileResponse
from sqlalchemy.ext.asyncio import AsyncSession

from core.deps import get_current_user, get_db
from models.user import User
from repository.user_repository import UserRepository
from schemas.auth import (
    ForgotPasswordRequest,
    LoginResponse,
    RefreshTokenRequest,
    RegisterResponse,
    ResetPasswordRequest,
    SendOtpRequest,
    VerifyOtpRequest,
)
from schemas.user import (
    GoogleAuthRequest,
    TokenResponse,
    UserCreate,
    UserLogin,
    UserResponse,
)
from services.auth_service import (
    create_access_token,
    create_refresh_token,
    decode_token,
    google_login,
    login_user,
    register_user,
)
from services.otp_service import send_otp, verify_otp
from services.reset_password_service import (
    request_password_reset,
    reset_password,
)

router = APIRouter(prefix="/auth", tags=["Auth"])


@router.post(
    "/register", response_model=RegisterResponse, status_code=status.HTTP_201_CREATED
)
async def register(user_data: UserCreate, db: AsyncSession = Depends(get_db)):
    return await register_user(user_data, db)


@router.post("/login", response_model=LoginResponse, status_code=status.HTTP_200_OK)
async def login(login_data: UserLogin, db: AsyncSession = Depends(get_db)):
    return await login_user(login_data, db)


@router.get("/me", response_model=UserResponse, status_code=status.HTTP_200_OK)
async def read_current_user(user: User = Depends(get_current_user)):
    return user


@router.post("/refresh", response_model=TokenResponse, status_code=status.HTTP_200_OK)
async def refresh(
    refresh_data: RefreshTokenRequest, db: AsyncSession = Depends(get_db)
):
    payload = decode_token(refresh_data.refresh_token, expected_type="refresh")
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired refresh token",
        )

    user_id = payload.get("sub")
    if not user_id:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token payload",
        )

    repo = UserRepository(db)
    user = await repo.get_user_by_id(UUID(user_id))

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )

    new_payload = {"sub": str(user.id)}
    access_token = create_access_token(new_payload)
    refresh_token = create_refresh_token(new_payload)

    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        token_type="bearer",
    )


@router.post("/logout", status_code=status.HTTP_200_OK)
async def logout(user: User = Depends(get_current_user)):
    return {"message": "Logged out successfully"}


@router.get("/forgot-password", include_in_schema=False)
def forget_password():
    return FileResponse("templates/forget-password.html")


@router.post("/forgot-password", status_code=status.HTTP_200_OK)
async def request_password(
    body: ForgotPasswordRequest, db: AsyncSession = Depends(get_db)
):
    return await request_password_reset(body.email, db)


@router.get("/reset-password")
def reset_password_page(token: str):
    return FileResponse("templates/password-reset.html")


@router.put("/reset-password")
async def update_password(
    body: ResetPasswordRequest, db: AsyncSession = Depends(get_db)
):
    return await reset_password(body.token, body.password, db)


@router.post("/google", response_model=LoginResponse, status_code=status.HTTP_200_OK)
async def google_auth(body: GoogleAuthRequest, db: AsyncSession = Depends(get_db)):
    return await google_login(body, db)


# --- OTP endpoints (NEW) ---


@router.post("/send-otp", status_code=status.HTTP_200_OK)
async def request_otp(body: SendOtpRequest, db: AsyncSession = Depends(get_db)):
    repo = UserRepository(db)
    user = await repo.get_user_by_email(body.email)
    if not user:
        raise HTTPException(status_code=404, detail="No account found with that email")
    return await send_otp(user.id, user.email, body.purpose, db)


@router.post("/verify-otp", response_model=LoginResponse, status_code=status.HTTP_200_OK)
async def confirm_otp(body: VerifyOtpRequest, db: AsyncSession = Depends(get_db)):
    repo = UserRepository(db)
    user = await repo.get_user_by_email(body.email)
    if not user:
        raise HTTPException(status_code=404, detail="No account found with that email")

    await verify_otp(user.id, body.otp, body.purpose, db)

    # re-fetch: verify_otp flips is_verified in the DB
    user = await repo.get_user_by_email(body.email)

    payload = {"sub": str(user.id)}
    access_token = create_access_token(payload)
    refresh_token = create_refresh_token(payload)

    return LoginResponse(
        message="Email verified successfully",
        user=user,
        tokens=TokenResponse(
            access_token=access_token,
            refresh_token=refresh_token,
            token_type="bearer",
        ),
    )
```

---

## 10. MODIFIED — `core/deps.py`

```python
from uuid import UUID

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from sqlalchemy.ext.asyncio import AsyncSession

from core.database import AsyncSessionLocal
from repository.user_repository import UserRepository
from services.auth_service import decode_token


async def get_db():
    async with AsyncSessionLocal() as session:
        yield session


security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
):
    token = credentials.credentials

    payload = decode_token(token, expected_type="access")

    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired authentication credentials",
        )

    user_id = payload.get("sub")
    if not user_id:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token payload",
        )

    repo = UserRepository(db)

    user = await repo.get_user_by_id(UUID(user_id))

    if not user or not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Inactive or Invalid user",
        )

    if not user.is_verified:  # NEW
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Email not verified",
        )

    return user
```

---

## 11. Database migration

Two schema changes: a new column on `users`, and a new `otp_verifications` table.

**Generate it** (if using Alembic):
```bash
alembic revision --autogenerate -m "add is_verified to users, add otp_verifications table"
alembic upgrade head
```

**Important — existing users**: `is_verified` defaults to `False` at the Python
level, but for a migration on an *existing* populated table you almost certainly
want existing rows to stay usable. In the generated migration, set the column
default explicitly and backfill:

```python
def upgrade():
    op.add_column(
        "users",
        sa.Column("is_verified", sa.Boolean(), nullable=False, server_default=sa.false()),
    )
    # Backfill: treat all pre-existing accounts as already verified,
    # so this feature doesn't lock out your current user base.
    op.execute("UPDATE users SET is_verified = true")

    op.create_table(
        "otp_verifications",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("user_id", postgresql.UUID(as_uuid=True), sa.ForeignKey("users.id"), nullable=False),
        sa.Column("otp_hash", sa.String(), nullable=False),
        sa.Column("purpose", sa.String(), nullable=False, server_default="email_verification"),
        sa.Column("expire_at", sa.TIMESTAMP(timezone=True), nullable=False),
        sa.Column("attempts", sa.Integer(), nullable=False, server_default="0"),
        sa.Column("used", sa.Boolean(), nullable=False, server_default=sa.false()),
        sa.Column("created_at", sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
    )


def downgrade():
    op.drop_table("otp_verifications")
    op.drop_column("users", "is_verified")
```

---

## 12. Env vars to add (`.env`)

```
OTP_EXPIRE_MINUTES=10
OTP_MAX_ATTEMPTS=5
```
(Both have safe defaults in `core/config.py` if omitted.)

---

## Summary of behavior after these changes

| Action | Before | After |
|---|---|---|
| Register with email/password | Tokens issued immediately | User created, OTP emailed, **no tokens** until verified |
| Login (unverified) | Tokens issued | **403** "Email not verified" |
| Login (verified) | Tokens issued | Tokens issued (unchanged) |
| `/auth/verify-otp` | — | Marks verified, **issues tokens** |
| `/auth/send-otp` | — | Resend OTP (also used for re-registration retries) |
| Google login, new email | Account created | Account created, `is_verified=True` (Google already vouched) |
| Google login, existing **local** email | **400 rejected** | Linked — `google_id` attached, `is_verified=True` |
| Any protected route with a stray/rolled-back-unverified token | Passed | **403** via `get_current_user` |
