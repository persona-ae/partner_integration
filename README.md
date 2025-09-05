# Persona Avatar Integration

Welcome! 🎉  

This repository provides everything you need to **embed and integrate the Persona Avatar pixel-streaming app** into your own mobile apps, web apps, or backend services. Our goal is to make it straightforward for partners to deliver rich, real-time avatar experiences to their users while keeping integration secure, reliable, and flexible.

---

## What is the Persona Avatar Pixel-Streaming App?

The Persona Avatar app is a low-latency, WebRTC-based experience that streams interactive avatars directly to your client. It is designed to:

- Render high-fidelity avatars in real time using Unreal Engine + Pixel Streaming.  
- Connect seamlessly to your backend systems using our APIs and secure tokens.  
- Run inside your own **mobile apps** (via WebViews), **websites** (via iframes or script tags), or any environment with a modern browser.  
- Provide hooks and callbacks so you can track sessions, customize behavior, and deliver personalized experiences.

---

## Integration Paths

There are **two main integration tracks**, depending on your needs:

1. **Client-side integration**  
   → [client_integration.md](client_integration.md)  
   How to embed the Persona Avatar app into your **mobile or web clients**. Covers:
   - Using WebViews (iOS/Android) or iframes.  
   - Passing authentication tokens.  
   - Receiving lifecycle callbacks (e.g., `session.started`, `session.ended`).  
   - Handling UX concerns such as permissions and errors.

2. **Server-side integration**  
   → [server_integration.md](server_integration.md)  
   REST API endpoints for accessing data from your backend. Covers:
   - JWT-based authentication with HMAC signatures.  
   - Listing available experiences and their capabilities.  
   - Retrieving user information and session history.  
   - Accessing detailed session transcripts and analytics.

---

## Who Is This For?

This documentation is intended for **partner developers and technical teams** who want to integrate Persona Avatar capabilities into their own products.  
If you are a product manager, integrator, or technical lead, you’ll find high-level architecture guidance as well as code snippets for practical implementation.

---

## Getting Help

- **Questions about integration?** Start with the relevant section above.  
- **Need support or keys?** Contact our team through your partner manager.  
- **Found an issue?** Please file a ticket with detailed logs, timestamps, and your partner ID (`iss`).  

---

We’re excited to see what you build with Persona Avatars. 🚀
