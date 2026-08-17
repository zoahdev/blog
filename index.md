---
layout: home
---

# zoahdev — engineering notes

Evidence-first engineering deep-dives on **DeepSeek Harness** and agent
runtimes: root causes, reproductions, mechanism-level verification, and
patches that are ready for upstream.

- Maintainer of the [DSH ecosystem map](https://github.com/zoahdev/dsh-ecosystem)
  (weekly editions) and a [42-patch upstream queue](https://github.com/zoahdev/dsh-docs/blob/main/docs/specs/upstream-patches.md).
- Author of [dsh-github-intelligence](https://github.com/zoahdev/dsh-github-intelligence)
  (196+ read-only tools, 16 ecosystems) and [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor).
- Every claim here comes with a reproducible trace; nothing is asserted without
  evidence.

## Posts

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) · {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}
