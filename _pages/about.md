---
permalink: /
title: "DevOps & Platform Engineering"
excerpt: "Kubernetes, GitOps, and cloud-native insights"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

# Hey there! 👋

I'm **Georg Novotny**, a Platform Engineer and **CKA/D-certified Kubernetes** enthusiast based in Vienna. I currently work at [MIC](https://www.mic-cust.com/) where I build and maintain cloud-native infrastructure, Helm Charts, CI/CD pipelines, and GitOps workflows.

## What I Do

🚀 **Platform Engineering** — Building reliable Kubernetes platforms with tools like ArgoCD, Vault, Cert-manager, and Kyverno

📦 **GitOps & Infrastructure as Code** — Managing clusters the declarative way with Helm, Kustomize, and the App of Apps pattern

🏠 **Homelab Enthusiast** — Running an over-engineered Kubernetes homelab that I document in my [GitOps blog series](/posts/2025/08/20/homelab-part1/)

🤖 **Robotics Enthusiast** — Teaching and tinkering with ROS-based autonomous vehicles and delivery robots. You can see more at [Robotics Content Lab](https://www.roboticscontentlab.com/).

## Background

Before diving into DevOps full-time, I spent several years in the **autonomous systems** world at [JKU](https://www.jku.at/en/intelligent-transport-systems/) and [UAS Technikum Wien](https://www.technikum-wien.at/). There I developed ROS-based autonomous vehicles, delivery robots, and search and rescue systems. I was also a part-time lecturer teaching mobile robotics and containerization for robotics applications.

This unique combination of **robotics + platform engineering** gives me a practical perspective on building reliable infrastructure for complex systems.

## Current Focus

- 🎓 Preparing for the AZ-104 certification
- 📖 Tinkering and learning with my self-hosted Kubernetes cluster
- 🔬 Working on improving my `ROS 2 - From Zero to Hero` book and container images

## Get in Touch

Check out my [blog posts](/year-archive/) to see what I'm working on, or explore my [projects](/portfolio/) and [publications](/publications/) for the full picture.

---

## 📝 Latest Blog Posts

<div class="recent-posts">
{% assign sorted_posts = site.posts | sort: 'date' | reverse %}
{% for post in sorted_posts limit:3 %}
<a href="{{ post.url }}" class="post-card">
  <div class="post-card__content">
    <span class="post-card__date">{{ post.date | date: "%B %d, %Y" }}</span>
    <h3 class="post-card__title">{{ post.title }}</h3>
    {% if post.tags.size > 0 %}
    <div class="post-card__tags">
      {% for tag in post.tags limit:3 %}
      <span class="post-card__tag">{{ tag }}</span>
      {% endfor %}
    </div>
    {% endif %}
  </div>
  <span class="post-card__arrow">→</span>
</a>
{% endfor %}
</div>

<a href="/year-archive/" class="view-all-posts">View All Posts →</a>

<style>
.recent-posts {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin: 1.5em 0;
}

.post-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px;
  background: var(--bg-secondary, #161b22);
  border: 1px solid var(--border-color, #30363d);
  border-radius: 12px;
  text-decoration: none !important;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.post-card::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 4px;
  background: var(--accent-gradient, linear-gradient(180deg, #58a6ff, #a371f7));
  opacity: 0;
  transition: opacity 0.3s ease;
}

.post-card:hover {
  border-color: var(--link-color, #58a6ff);
  transform: translateX(4px);
  box-shadow: 0 4px 20px var(--shadow-color, rgba(0, 0, 0, 0.3));
}

.post-card:hover::before {
  opacity: 1;
}

.post-card__content {
  flex: 1;
}

.post-card__date {
  font-size: 0.8em;
  color: var(--text-muted, #8b949e);
  margin-bottom: 4px;
  display: block;
}

.post-card__title {
  margin: 0 0 8px 0;
  font-size: 1.1em;
  color: var(--text-primary, #f0f6fc);
  font-weight: 600;
}

.post-card__tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.post-card__tag {
  font-size: 0.75em;
  padding: 3px 10px;
  background: rgba(88, 166, 255, 0.1);
  border: 1px solid rgba(88, 166, 255, 0.2);
  border-radius: 12px;
  color: var(--link-color, #58a6ff);
}

.post-card__arrow {
  font-size: 1.5em;
  color: var(--text-muted, #8b949e);
  transition: all 0.3s ease;
}

.post-card:hover .post-card__arrow {
  color: var(--link-color, #58a6ff);
  transform: translateX(4px);
}

.view-all-posts {
  display: inline-block;
  margin-top: 10px;
  padding: 12px 24px;
  background: var(--bg-tertiary, #21262d);
  border: 1px solid var(--border-color, #30363d);
  border-radius: 8px;
  color: var(--link-color, #58a6ff) !important;
  text-decoration: none !important;
  font-weight: 500;
  transition: all 0.3s ease;
}

.view-all-posts:hover {
  background: var(--link-color, #58a6ff);
  color: #fff !important;
  border-color: var(--link-color, #58a6ff);
}
</style>
