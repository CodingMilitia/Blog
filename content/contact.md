---
layout: page
title: Contact
permalink: /contact/
---

If you can't get in touch through one of the channels listed in the footer, feel free to use this form.

I'll try to get back to you as soon as possible.

<form name="contact" method="POST" data-netlify="true" netlify-honeypot="bot-field">
    <div class="honeypot-hidden">
        <label>Don’t fill this out if you're human: <input name="bot-field" /></label>
    </div>
    <div class="field">
        <label class="label">Your Name:</label>
        <input type="text" name="name" class="input" />
    </div>
    <div class="field">
        <label class="label">Your Email:</label>
        <input type="email" name="email" class="input" />
    </div>
    <div class="field">
        <label class="label">Message:</label>
        <textarea name="message" class="textarea"></textarea>
    </div>
    <div class="control">
        <button type="submit" class="button is-link">Send</button>
    </div>
</form>
