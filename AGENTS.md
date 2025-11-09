# AI Agent Guidelines for Jekyll Blog Project

## Project Overview

This is a Jekyll-based blog hosted on GitHub Pages. The blog is built using Ruby and Jekyll static site generator.

**Project URL**: https://zzfd.github.io
**Local Dev Server**: http://127.0.0.1:4000

## Technical Stack

- **Ruby Version**: 3.3.0 (managed via rbenv)
- **Jekyll Version**: 3.10.0
- **GitHub Pages Gem**: 232
- **Bundle Manager**: Bundler 2.7.2
- **Platform**: macOS ARM64 (darwin24)

## Project Structure

```
.
├── _config.yml          # Jekyll configuration
├── _posts/              # Blog posts directory
├── _includes/           # Reusable components
├── _layouts/            # Page layouts
├── _site/               # Generated site (do not edit)
├── assets/              # Static assets (images, CSS, JS)
├── Gemfile              # Ruby dependencies
├── Gemfile.lock         # Locked dependency versions
└── .ruby-version        # rbenv Ruby version file
```

## Blog Post Management

### Creating New Posts

1. **File Naming Convention**:
   - Format: `YYYY-MM-DD-标题.md`
   - Example: `2025-11-10-重启博客.md`
   - Place in `_posts/` directory

2. **Front Matter Template**:
   ```yaml
   ---
   layout: post
   title: "文章标题"
   date: YYYY-MM-DD HH:MM:SS +0800
   categories: 分类名
   tags: 标签1 标签2
   ---
   ```

3. **Common Categories**:
   - `所学` - Technical learning
   - `所想` - Thoughts and reflections

4. **Post Footer**:
   Always include a reference link at the end:
   ```markdown
   原文地址：<a href="https://zzfd.github.io/YYYY/MM/DD/文章标题">文章标题</a>
   ```

### Post Content Guidelines

- Use Markdown syntax
- For code blocks, use Jekyll's highlight tags:
  ```liquid
  {% highlight language %}
  code here
  {% endhighlight %}
  ```
- Supported languages: `json`, `js`, `wxml`, `wxss`, `html`, `css`, `python`, etc.
- For images: `<img src="{{site.baseurl}}/assets/res/filename.png" alt="description"/>`

## Development Workflow

### Starting the Development Server

```bash
bundle exec jekyll serve
```

- Server runs on `http://127.0.0.1:4000`
- Auto-regeneration is enabled
- Press Ctrl+C to stop

### Installing Dependencies

```bash
bundle install
```

### Checking Ruby Version

```bash
ruby --version  # Should be 3.3.0
rbenv local     # Check local version
```

## Common Tasks for AI Assistants

### Task 1: Create a New Blog Post

**Steps**:
1. Ask user for:
   - Post title (Chinese or English)
   - Content
   - Category (default: `所学`)
   - Tags (optional)
2. Generate filename with current date: `YYYY-MM-DD-title.md`
3. Create file in `_posts/` with proper front matter
4. Include reference link at the end
5. Verify Jekyll auto-regenerated the site (check server output)

**Example**:
```markdown
---
layout: post
title: "我的新文章"
date: 2025-11-10 12:00:00 +0800
categories: 所学
tags: JavaScript Vue
---

文章内容...

原文地址：<a href="https://zzfd.github.io/2025/11/10/我的新文章">我的新文章</a>
```

### Task 2: Update Existing Post

**Steps**:
1. Search for post in `_posts/` directory
2. Read existing content
3. Make requested changes
4. Preserve front matter and reference link
5. Verify regeneration

### Task 3: Add Images/Assets

**Steps**:
1. Save images to `assets/res/` or `assets/images/`
2. Reference in posts: `{{site.baseurl}}/assets/res/filename.png`
3. Use appropriate alt text for accessibility

### Task 4: Fix Configuration Issues

**Known Issues**:
1. **Deprecation**: Change `gems:` to `plugins:` in `_config.yml`
2. **Liquid Warnings**: Fix JavaScript syntax in Liquid templates (use Liquid filters instead)
3. **GitHub API**: Set JEKYLL_GITHUB_TOKEN for metadata (optional)

## Important Notes

### DO:
- ✓ Always use UTF-8 encoding for Chinese characters
- ✓ Test posts locally before committing
- ✓ Follow existing post format and structure
- ✓ Use proper YAML front matter
- ✓ Include reference links at post end
- ✓ Check Jekyll regeneration output for errors

### DON'T:
- ✗ Edit files in `_site/` directory (auto-generated)
- ✗ Modify `Gemfile.lock` manually
- ✗ Change Ruby version without testing
- ✗ Use raw JavaScript syntax in Liquid templates
- ✗ Forget to include proper front matter

## Troubleshooting

### Jekyll Server Won't Start
```bash
# Check Ruby version
ruby --version

# Reinstall dependencies
bundle install

# Check for syntax errors
bundle exec jekyll build --verbose
```

### Post Not Appearing
1. Check filename format: `YYYY-MM-DD-title.md`
2. Verify front matter is valid YAML
3. Check date is not in future
4. Look for errors in server output

### Ruby Version Issues
```bash
# Set local Ruby version
rbenv local 3.3.0

# Verify
rbenv version
```

## Git Workflow

### Committing Changes
```bash
git add .
git commit -m "Add new post: 文章标题"
git push origin main
```

### GitHub Pages Deployment
- Automatic on push to `main` branch
- Check deployment status on GitHub
- May take 1-2 minutes to update

## Reference Links

- Jekyll Documentation: https://jekyllrb.com/docs/
- GitHub Pages: https://pages.github.com/
- Liquid Templates: https://shopify.github.io/liquid/
- Markdown Guide: https://www.markdownguide.org/

## AI Assistant Behavior Guidelines

1. **Be Proactive**: Suggest improvements to post structure, SEO, or formatting
2. **Validate Input**: Check dates, categories, and file naming
3. **Verify Operations**: Always check Jekyll regeneration output
4. **Preserve History**: Don't delete old posts without user confirmation
5. **Follow Conventions**: Maintain consistency with existing posts
6. **Error Handling**: Report Jekyll errors clearly and suggest fixes
7. **Chinese Support**: Ensure proper UTF-8 encoding for Chinese content

## Example Interactions

### Example 1: User asks to create a post
```
User: "写一篇关于React Hooks的博客"
AI:
1. Creates file: 2025-11-10-React-Hooks学习笔记.md
2. Uses category: 所学
3. Adds tags: React JavaScript
4. Structures content with proper headers
5. Includes code examples with highlight tags
6. Adds reference link
7. Confirms Jekyll regenerated successfully
```

### Example 2: User wants to update config
```
User: "修复配置警告"
AI:
1. Reads _config.yml
2. Changes 'gems:' to 'plugins:'
3. Explains the change
4. Restarts Jekyll server
5. Verifies warning is gone
```

## Version History

- **v1.0** (2025-11-10): Initial guidelines created
  - Ruby upgraded from 3.1.0 to 3.3.0
  - Fixed activesupport compatibility issues
  - Documented blog post creation workflow

---

**Last Updated**: 2025-11-10
**Maintainer**: AI Assistant (Claude Code)
