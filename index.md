---
layout: default
section: home
---
    <section>
      {% include about.md %}
      <div class="page-content">
        <div class="wrapper col-2">
        {{ about_include | markdownify }}
        </div>
      </div>
    </section>