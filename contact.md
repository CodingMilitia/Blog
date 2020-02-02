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
    <div class="form-group">
        <label>Your Name:</label>
        <input type="text" name="name" class="form-control" />
    </div>
    <div class="form-group">
        <label>Your Email:</label>
        <input type="email" name="email" class="form-control" />
    </div>
    <div class="form-group">
        <label>Message:</label>
        <textarea name="message" class="form-control"></textarea>
    </div>
    <div class="form-group">
        <button type="submit">Send</button>
    </div>
</form>
