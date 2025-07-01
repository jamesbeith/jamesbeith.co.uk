---
title: "How to use reserved domains in tests to prevent DNS queries"
date: "2025-07-01T16:45:00+10:00"
---

When writing tests I use the `example.com` domain for any URLs, email addresses, and similar. The reason: if my test unintentionally made a real request, or sent a real email, then it would resolve to a special-use domain. The Internet Assigned Numbers Authority (IANA) reserves the `example.com` domain for documentation purposes. Better that the request resolves there than to one that’s privately owned, such as `test.com`.

While reviewing some work recently I came across a comment from [Ben](https://github.com/bcdickinson) about his suggestion to use the `.invalid` top-level domain. I learned that DNS prohibits installing certain TLDs like `.example`, `.invalid`, `.localhost`, and `.test`. Using test data such as `john@example.invalid` is safer than `john@example.com` because requests should never even reach DNS servers. While all four TLDs receive special handling, `.invalid` is the most non-resolving. For the technical differences between them, see sections 6.2-6.5 of [RFC 6761](https://www.rfc-editor.org/rfc/rfc6761.html#section-6.2).
