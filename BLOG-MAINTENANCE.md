# Blog Maintenance Guide

## 📝 How to Add New Entries to Your Learning Log

Every time you write a new learning entry, you need to update **3 files**:

### 1. Create the Entry File
Use `entry-template.html` to create your new entry.

**Example filename:** `2025-11-03-learning-linq.html`

### 2. Update index.html (Home Page)
Add your entry to the "Recent Activity" section at the **top** (newest first).

**Location in file:** Look for `<!-- Entry 1 - Most Recent -->`

**Copy this template:**
```html
<div style="border: 1px solid var(--border-color); border-radius: 8px; padding: 1.5rem; background: white; margin-bottom: 1.5rem;">
  <h3 style="margin-top: 0;">
    <a href="2025-11-03-learning-linq.html">Learning LINQ Fundamentals</a>
  </h3>
  <p class="metadata">📅 November 3, 2025 | 🏷️ 
    <span class="tag">C#</span>
    <span class="tag">LINQ</span>
  </p>
  <p>Today I learned the basics of LINQ queries and how to filter collections efficiently using Where, Select, and OrderBy methods.</p>
  <a href="2025-11-03-learning-linq.html">Read more →</a>
</div>
```

**Important:** 
- Keep only the 5-10 most recent entries on the home page
- Older entries should be removed from home page but kept in `all-entries.html`

### 3. Update all-entries.html (Archive Page)
Add your entry to the appropriate month section.

**Location in file:** Find the month section (e.g., `<h2>📅 November 2025</h2>`)

**Copy this template:**
```html
<div style="border: 1px solid var(--border-color); border-radius: 8px; padding: 1.5rem; background: white; margin-bottom: 1.5rem;">
  <div style="display: flex; justify-content: space-between; align-items: start; flex-wrap: wrap; gap: 1rem;">
    <div style="flex: 1; min-width: 250px;">
      <h3 style="margin-top: 0;">
        <a href="2025-11-03-learning-linq.html">Learning LINQ Fundamentals</a>
      </h3>
      <p class="metadata">📅 November 3, 2025</p>
    </div>
    <div class="tags" style="margin: 0;">
      <span class="tag">C#</span>
      <span class="tag">LINQ</span>
    </div>
  </div>
  <p>Today I learned the basics of LINQ queries and how to filter collections efficiently.</p>
  <div style="display: flex; gap: 1rem; align-items: center;">
    <a href="2025-11-03-learning-linq.html" style="color: var(--primary-color); font-weight: 600;">Read full entry →</a>
    <span style="color: var(--text-secondary); font-size: 0.9rem;">Related: <a href="csharp-dotnet.html">C# & .NET</a></span>
  </div>
</div>
```

**Also update the category sections** at the bottom of `all-entries.html`:
- Add to the appropriate category (QA Testing, Development, Projects, or Learning)

### 4. Optional: Update the Relevant Topic Page
Add a link to your entry on the topic page (e.g., `csharp-dotnet.html`).

**Location:** Under the "📚 Learning Entries" section

```html
<div style="border: 1px solid var(--border-color); border-radius: 8px; padding: 1.5rem; background: white; margin-bottom: 1rem;">
  <h3 style="margin-top: 0;">
    <a href="2025-11-03-learning-linq.html">Learning LINQ Fundamentals</a>
  </h3>
  <p class="metadata">📅 November 3, 2025 | 🏷️ 
    <span class="tag">C#</span>
    <span class="tag">LINQ</span>
  </p>
  <p>Brief description.</p>
  <a href="2025-11-03-learning-linq.html">Read more →</a>
</div>
```

## 📊 Update Stats

### On index.html:
Change the "Entries Written" number:
```html
<h3 style="margin: 0; color: var(--success-color);">2</h3> <!-- Increase this -->
<p style="margin: 0.5rem 0 0 0;">Entries Written</p>
```

### On all-entries.html:
Update the stats at the top:
```html
<h3 style="margin: 0; color: var(--primary-color);">2</h3> <!-- Total entries -->
```

## 🗓️ Starting a New Month

When you write your first entry in a new month:

### In all-entries.html:
1. Add a new month section **above** the previous month
2. Update the sidebar with the new month

```html
<div class="section" id="december-2025">
  <h2>📅 December 2025</h2>
  
  <!-- Your entries here -->
</div>
```

### In the sidebar:
```html
<h3 style="font-size: 1rem;">🗓️ Browse by Date</h3>
<ul>
  <li><a href="#december-2025">December 2025</a></li>
  <li><a href="#november-2025">November 2025</a></li>
</ul>
```

## ✅ Quick Checklist for Each New Entry

- [ ] Create entry file (e.g., `2025-11-03-topic.html`)
- [ ] Add to index.html (Recent Activity, top of list)
- [ ] Add to all-entries.html (in month section)
- [ ] Add to all-entries.html (in category section)
- [ ] Update entry count on index.html
- [ ] Update entry count on all-entries.html
- [ ] Optional: Add to relevant topic page
- [ ] Commit and push to GitHub

## 🎨 Customization Tips

### Change how many entries show on home page:
- Show more: Keep more entry blocks in the Recent Activity section
- Show less: Remove older entry blocks (but keep them in all-entries.html)

### Featured entries:
You can add a special style to highlight important entries:
```html
<div style="border: 2px solid var(--primary-color); border-radius: 8px; padding: 1.5rem; background: #f0f0ff; margin-bottom: 1.5rem;">
  <span style="background: var(--primary-color); color: white; padding: 0.25rem 0.75rem; border-radius: 12px; font-size: 0.8rem; font-weight: 600;">FEATURED</span>
  <!-- Rest of entry -->
</div>
```

## 💡 Pro Tips

1. **Consistent naming:** Use date prefix for all entries (YYYY-MM-DD-topic.html)
2. **Keep descriptions short:** 2-3 sentences max on index page
3. **Use good tags:** Makes filtering easier later
4. **Link related entries:** Add "Related" links to connect similar topics
5. **Regular updates:** Easier to maintain if you update as you write
6. **Backup before editing:** GitHub saves versions, but be careful

## 🆘 Common Mistakes

❌ **Don't forget to:**
- Update the stats
- Add to all-entries.html
- Keep dates consistent
- Update category sections

✅ **Remember to:**
- Test links work
- Check mobile view
- Proofread dates
- Keep newest entries at the top

---

Happy blogging! 📝
