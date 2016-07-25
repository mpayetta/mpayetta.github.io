---
layout: page
title: Contact
permalink: /contact/
comments: false
---

Feel free to reach me anytime at <a href="mailto:mauricio@bithive.io">mauricio@bithive.io</a> or any of the social links below.

<ul class="social about">
    {% if site.twitter_url != '' %}<li><a href="{{ site.twitter_url }}" class="twitter" target="_blank"><i class="fa fa-twitter"></i></a></li>{% endif %}
    {% if site.facebook_url != '' %}<li><a href="{{ site.facebook_url }}" class="facebook" target="_blank"><i class="fa fa-facebook"></i></a></li>{% endif %}
    {% if site.github_url != '' %}<li><a href="{{ site.github_url }}" class="github" target="_blank"><i class="fa fa-github-alt"></i></a></li>{% endif %}
    {% if site.instagram_url != '' %}<li><a href="{{ site.instagram_url }}" class="instagram" target="_blank"><i class="fa fa-instagram"></i></a></li>{% endif %}
    {% if site.linkedin_url != '' %}<li><a href="{{ site.linkedin_url }}" class="linkedin" target="_blank"><i class="fa fa-linkedin"></i></a></li>{% endif %}
    <li><a href="{{ site.baseurl }}/feed.xml" class="rss" target="_blank"><i class="fa fa-rss"></i></a></li>
</ul>
