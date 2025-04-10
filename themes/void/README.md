# Void - A Clean Modern Hugo Blog Theme

Void is a clean, modern Hugo blog theme built with [Tailwind CSS](https://tailwindcss.com/). It's designed for personal blogs with a focus on content presentation and reading experience.

<!-- Screenshot will be added in the future -->
<!-- ![Void Theme Preview](./static/images/screenshot.png) -->

## Features

- 🎨 Built with Tailwind CSS for a clean, modern design
- 📱 Fully responsive design, works on all devices
- 🔍 SEO optimized
- 💡 Light mode with focus on reading experience
- 📝 Code highlighting with customizable styles
- 🏷️ Support for tags and categories
- 📊 Reading time estimation
- 🌐 Social media integration
- 📋 Code block copy functionality
- 🔤 Language labels for code blocks

## Installation

### Method 1: As a Git Submodule (Recommended)

```bash
cd yourHugoSite
git submodule add https://github.com/Daucloud/hugo-theme-void themes/void
```

### Method 2: Direct Download

1. Download the [latest release](https://github.com/Daucloud/hugo-theme-void/releases)
2. Extract to the `themes/void` directory
3. Set the theme in your Hugo configuration file: `theme = "void"`

## Quick Start

Create a new site and apply the Void theme:

```bash
hugo new site mysite
cd mysite
git init
git submodule add https://github.com/Daucloud/hugo-theme-void themes/void
echo 'theme = "void"' >> hugo.toml
hugo server -D
```

## Configuration

Add the following configuration options to your `hugo.toml` (or `config.toml`):

```toml
baseURL = 'https://example.org/'
languageCode = 'en-US'
title = 'Your Site Title'
theme = "void"

# Social media links
[params]
  [params.social]
    github = "https://github.com/yourusername"
    twitter = "https://twitter.com/yourusername"
    email = "your.email@example.com"
  [params.avatar]
    url = "https://example.com/your-avatar.jpg"

# Main menu
[[menus.main]]
name = 'Home'
pageRef = '/'
weight = 10

[[menus.main]]
name = 'Posts'
pageRef = '/posts'
weight = 20

[[menus.main]]
name = 'Tags'
pageRef = '/tags'
weight = 30

[[menus.main]]
name = 'About'
pageRef = '/about'
weight = 40
```

## Development

If you want to modify the theme, you need to install Node.js and npm. Then:

```bash
cd themes/void
npm install
npm run dev # Development mode with auto CSS compilation
```

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgements

- [Tailwind CSS](https://tailwindcss.com/) - Utility-first CSS framework
- [Hugo](https://gohugo.io/) - The world's fastest framework for building websites
