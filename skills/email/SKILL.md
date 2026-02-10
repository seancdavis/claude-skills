---
name: email
description: Transactional email patterns for Netlify-hosted applications using Resend. Use when implementing email notifications, password resets, or any server-side email sending. Covers setup, template patterns, and integration with Netlify Functions.
---

# Email

Transactional email patterns using Resend.

---

## Provider Choice

**Recommended: Resend**

- Developer-friendly API
- Good free tier (3,000 emails/month)
- Works well with serverless
- Simple SDK
- React email templates supported

Other options: SendGrid, Postmark, AWS SES

---

## Setup

### Install Resend

```bash
npm install resend
```

### Store API Key

```bash
netlify env:set --secret RESEND_API_KEY "re_..."
```

### Configure Domain (Production)

In Resend dashboard:
1. Add your domain
2. Add DNS records
3. Verify domain

For development, use Resend's test mode or `onboarding@resend.dev`.

---

## Basic Email Sending

```typescript
import { Resend } from "resend";

const resend = new Resend(process.env.RESEND_API_KEY);

async function sendEmail({
  to,
  subject,
  html,
}: {
  to: string;
  subject: string;
  html: string;
}) {
  const { data, error } = await resend.emails.send({
    from: "App Name <notifications@yourdomain.com>",
    to,
    subject,
    html,
  });

  if (error) {
    console.error("Email error:", error);
    throw new Error("Failed to send email");
  }

  return data;
}
```

---

## In Netlify Functions

```typescript
// netlify/functions/send-notification.ts
import type { Context, Config } from "@netlify/functions";
import { Resend } from "resend";
import { requireAuth } from "./_shared/auth";

const resend = new Resend(process.env.RESEND_API_KEY);

export default async (request: Request, context: Context) => {
  if (request.method !== "POST") {
    return new Response("Method not allowed", { status: 405 });
  }

  const auth = await requireAuth(request);
  if (!auth.authenticated) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { recipientEmail, message } = await request.json();

  try {
    const { data, error } = await resend.emails.send({
      from: "App <notifications@yourdomain.com>",
      to: recipientEmail,
      subject: "New Notification",
      html: `<p>${message}</p>`,
    });

    if (error) {
      console.error("Email error:", error);
      return Response.json({ error: "Failed to send email" }, { status: 500 });
    }

    return Response.json({ success: true, id: data?.id });
  } catch (error) {
    console.error("Email error:", error);
    return Response.json({ error: "Failed to send email" }, { status: 500 });
  }
};

export const config: Config = {
  path: "/api/send-notification",
  method: "POST",
};
```

---

## In Astro API Routes

```typescript
// src/pages/api/contact.ts
import type { APIRoute } from "astro";
import { Resend } from "resend";
import { logger } from "../../lib/logger";

const log = logger.scope("EMAIL");
const resend = new Resend(process.env.RESEND_API_KEY);

export const POST: APIRoute = async ({ request, redirect }) => {
  const formData = await request.formData();
  const name = formData.get("name")?.toString();
  const email = formData.get("email")?.toString();
  const message = formData.get("message")?.toString();

  if (!name || !email || !message) {
    return redirect("/contact?message=validation_error", 302);
  }

  try {
    const { error } = await resend.emails.send({
      from: "Contact Form <notifications@yourdomain.com>",
      to: "you@example.com",
      replyTo: email,
      subject: `Contact from ${name}`,
      html: `
        <h2>New Contact Form Submission</h2>
        <p><strong>Name:</strong> ${name}</p>
        <p><strong>Email:</strong> ${email}</p>
        <p><strong>Message:</strong></p>
        <p>${message}</p>
      `,
    });

    if (error) {
      log.error("Failed to send contact email:", error);
      return redirect("/contact?message=email_failed", 302);
    }

    log.info("Contact email sent from:", email);
    return redirect("/contact?message=email_sent", 302);
  } catch (error) {
    log.error("Email error:", error);
    return redirect("/contact?message=email_failed", 302);
  }
};
```

---

## Email Templates

### Simple HTML Template

```typescript
function welcomeEmail(name: string): string {
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <style>
          body { font-family: sans-serif; line-height: 1.6; color: #333; }
          .container { max-width: 600px; margin: 0 auto; padding: 20px; }
          .button {
            display: inline-block;
            padding: 12px 24px;
            background: #3b82f6;
            color: white;
            text-decoration: none;
            border-radius: 6px;
          }
        </style>
      </head>
      <body>
        <div class="container">
          <h1>Welcome, ${name}!</h1>
          <p>Thanks for signing up. We're excited to have you.</p>
          <p>
            <a href="https://yourapp.com/dashboard" class="button">
              Get Started
            </a>
          </p>
        </div>
      </body>
    </html>
  `;
}
```

### Template Utility

```typescript
// src/lib/email-templates.ts
export const templates = {
  welcome: (name: string) => ({
    subject: "Welcome to App Name!",
    html: welcomeEmail(name),
  }),

  passwordReset: (resetUrl: string) => ({
    subject: "Reset Your Password",
    html: `
      <p>Click the link below to reset your password:</p>
      <p><a href="${resetUrl}">${resetUrl}</a></p>
      <p>This link expires in 1 hour.</p>
    `,
  }),

  notification: (title: string, body: string) => ({
    subject: title,
    html: `<h2>${title}</h2><p>${body}</p>`,
  }),
};
```

---

## React Email Templates (Optional)

For complex templates, use React Email:

```bash
npm install @react-email/components react-email
```

```tsx
// emails/welcome.tsx
import { Html, Head, Body, Container, Text, Button } from "@react-email/components";

interface WelcomeEmailProps {
  name: string;
}

export default function WelcomeEmail({ name }: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Body style={{ fontFamily: "sans-serif" }}>
        <Container>
          <Text>Welcome, {name}!</Text>
          <Button href="https://yourapp.com/dashboard">
            Get Started
          </Button>
        </Container>
      </Body>
    </Html>
  );
}
```

```typescript
import { render } from "@react-email/render";
import WelcomeEmail from "../emails/welcome";

const html = await render(WelcomeEmail({ name: "John" }));
```

---

## Error Handling

```typescript
try {
  const { data, error } = await resend.emails.send({ ... });

  if (error) {
    // Resend-specific error
    console.error("Resend error:", error.name, error.message);

    if (error.name === "validation_error") {
      // Invalid email address
    } else if (error.name === "rate_limit_exceeded") {
      // Too many emails
    }

    return { success: false, error: error.message };
  }

  return { success: true, id: data?.id };
} catch (error) {
  // Network or other error
  console.error("Email error:", error);
  return { success: false, error: "Failed to send email" };
}
```

---

## Development/Testing

### Using Test Mode

In development, Resend sends to a test endpoint:

```typescript
const resend = new Resend(process.env.RESEND_API_KEY);

// Will show in Resend dashboard but not actually send
await resend.emails.send({
  from: "onboarding@resend.dev", // Use Resend's test address
  to: "test@example.com",
  subject: "Test",
  html: "<p>Test</p>",
});
```

### Logging Instead of Sending

```typescript
const isDev = process.env.NODE_ENV === "development";

if (isDev) {
  console.log("Would send email:", { to, subject, html });
  return { success: true, id: "dev-mock" };
}

return await resend.emails.send({ ... });
```

---

## Anti-Patterns

- **Sending from unverified domains** - Emails will be marked as spam
- **No error handling** - Email sending can fail; handle gracefully
- **Exposing email endpoints** - Require authentication
- **Sending sensitive data in plain text** - Use secure links instead
- **No rate limiting** - Prevent abuse of email endpoints

---

## Related Skills

- `netlify-functions` - API endpoint implementation
- `forms` - Contact form handling
- `feedback` - Showing email status to users
- `environment-variables` - Storing API keys
