<script setup lang="ts">
import { ref, onMounted } from 'vue';
import TurndownService from 'turndown';

const status = ref('Ready to convert');
const isConverting = ref(false);
const isExtensionContext = ref(true);
const removeBeforeHeading = ref(false);
const articleOnly = ref(false);

const turndownService = new TurndownService({
  headingStyle: 'atx',
  codeBlockStyle: 'fenced',
  bulletListMarker: '-',
  hr: '---',
  strongDelimiter: '**',
  emDelimiter: '_',
  linkStyle: 'inlined',
  preformattedCode: true,
  fence: '```',
  blankReplacement: function (_content: string, node: Node) {
    return (node as HTMLElement).tagName === 'DIV' || (node as HTMLElement).tagName === 'P' ? '\n' : '';
  },
  keepReplacement: function (content: string, node: Node) {
    const isBlock = ['DIV', 'P', 'PRE', 'CODE'].includes((node as HTMLElement).tagName);
    return isBlock ? '\n' + content + '\n' : content;
  }
});

// Custom rules for better formatting
turndownService.addRule('codeBlocks', {
  filter: ['pre', 'code'],
  replacement: (content: string, node: Node) => {
    const element = node as HTMLElement;
    if (element.tagName === 'CODE' && element.parentElement?.tagName === 'PRE') {
      const language = element.className?.match(/language-(\w+)/)?.[1] ||
        element.parentElement?.className?.match(/language-(\w+)/)?.[1] || '';
      return `\n\n\`\`\`${language}\n${content.trim()}\n\`\`\`\n\n`;
    }
    const language = element.className?.match(/language-(\w+)/)?.[1] || '';
    if (!element.parentElement?.matches('pre')) {
      return `\`${content.trim()}\``;
    }
    return `\n\n\`\`\`${language}\n${content.trim()}\n\`\`\`\n\n`;
  }
});

turndownService.addRule('cleanSpacing', {
  filter: ['p', 'div', 'section'],
  replacement: (content: string) => {
    return content.trim() ? '\n\n' + content.trim() + '\n\n' : '\n';
  }
});

turndownService.addRule('headings', {
  filter: ['h1', 'h2', 'h3', 'h4', 'h5', 'h6'],
  replacement: (content: string, node: Node) => {
    const level = Number((node as HTMLElement).tagName.charAt(1));
    const hashes = '#'.repeat(level);
    return `\n\n${hashes} ${content}\n\n`;
  }
});

async function getPageContent() {
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  if (!tab.id) throw new Error('No active tab found');

  const articleOnlyValue = articleOnly.value;

  const [result] = await chrome.scripting.executeScript({
    target: { tabId: tab.id },
    func: (articleOnly) => {
      function cleanNode(node: Element | Document): Element | Document {
        const unwanted = node.querySelectorAll([
          // Technical elements
          'script, style, link, meta, noscript, iframe, svg',

          // Navigation elements outside articles
          'nav, [role="navigation"], [role="complementary"], [role="banner"]',
          ':not(article) > header, :not(article) > footer, aside',

          // Common UI patterns (case-insensitive)
          '[class*="nav-"], [class*="menu-"], [class*="sidebar"]',
          '[class*="header" i]:not(article *), [class*="footer" i]:not(article *)',
          '[class*="cookie" i], [id*="cookie" i]',
          '[class*="banner" i], [id*="banner" i]',

          // Social and sharing elements
          '[class*="social" i], [class*="share" i]',

          // Promotional and advertising content
          '[class*="ad-" i], [class*="ads-" i], [class*="advert" i]',
          '[id*="ad-" i], [id*="ads-" i], [id*="advert" i]',
          '[data-ad], [data-ads], [data-advertisement]',

          // Popups and overlays
          '[class*="popup" i], [class*="modal" i], [class*="overlay" i]',

          // Related content and widgets
          '[class*="related" i], [class*="widget" i], [class*="recommended" i]'
        ].join(','));

        unwanted.forEach((el: Element) => el.remove());

        const elements = node.getElementsByTagName('*');
        for (const el of elements) {
          const attrs = el.attributes;
          for (let i = attrs.length - 1; i >= 0; i--) {
            const attr = attrs[i];
            if (attr.name.startsWith('on') ||
              attr.name === 'style' ||
              attr.value.includes('javascript:')) {
              el.removeAttribute(attr.name);
            }
          }
        }

        return node;
      }

      const doc = document.cloneNode(true) as Document;

      if (articleOnly) {
        const article = doc.querySelector(
          'article, ' +
          '[role="article"], ' +
          'main article, ' +
          '[itemprop="articleBody"], ' +
          '.post-content, ' +
          '.article-content, ' +
          '.entry-content, ' +
          '#article-content, ' +
          '.article__content, ' +
          '.post__content, ' +
          '[class*="article-body"], ' +
          '[class*="post-body"]'
        );

        if (article) {
          cleanNode(article);
          return article.outerHTML;
        }

        const mainContent = doc.querySelector('main, [role="main"]');
        if (mainContent) {
          cleanNode(mainContent);
          return mainContent.outerHTML;
        }

        return null;
      }

      cleanNode(doc.documentElement);
      return doc.documentElement.outerHTML;
    },
    args: [articleOnlyValue]
  });

  if (!result.result) {
    throw new Error('No content found to convert');
  }

  return result.result;
}

async function convertToMarkdown(articleOnlyMode = false) {
  if (!chrome?.tabs) {
    status.value = 'Error: Extension APIs not available';
    return;
  }

  try {
    isConverting.value = true;
    status.value = 'Converting...';

    articleOnly.value = articleOnlyMode;
    const html = await getPageContent();
    /* chrome.scripting.executeScript({
      target: { tabId: (await chrome.tabs.query({ active: true, currentWindow: true }))[0].id! },
      func: (html) => {
        console.log('HTML before conversion:', html);
      },
      args: [html]
    }); */

    let markdown = turndownService.turndown(html as string);

    /* chrome.scripting.executeScript({
      target: { tabId: (await chrome.tabs.query({ active: true, currentWindow: true }))[0].id! },
      func: (markdown) => {
        console.log('Markdown after conversion:', markdown);
      },
      args: [markdown]
    }); */

    const [currentTab] = await chrome.tabs.query({ active: true, currentWindow: true });

    if (articleOnlyMode) {
      markdown = `# ${currentTab.title || 'Article'}\n\n${markdown}`;
    } else if (!markdown.trim().startsWith('#')) {
      const headingMatch = markdown.match(/^[\s\S]*?(#{1,6}\s+[^\n]+)/);

      if (headingMatch) {
        if (removeBeforeHeading.value) {
          markdown = markdown.slice(markdown.indexOf(headingMatch[1]));
        }
      } else {
        markdown = `# ${currentTab.title || 'Page'}\n\n${markdown}`;
      }
    }

    markdown = markdown
      // Remove reference-style links at the end of the document
      .replace(/\n\s*\[[^\]]+\]:\s*http[^\n]+/g, '')
      // Remove multiple consecutive blank lines
      .replace(/\n{3,}/g, '\n\n')
      // Remove trailing whitespace
      .replace(/[ \t]+\n/g, '\n')
      // Ensure consistent heading spacing
      .replace(/\n(#{1,6} [^\n]+)/g, '\n\n$1\n\n')
      // Clean up list formatting
      .replace(/\n\s*[-*+]\s+/g, '\n- ')
      .replace(/(\n- [^\n]+)(\n(?!-))/g, '$1\n$2')
      // Remove empty links
      .replace(/\[([^\]]+)\]\[\]/g, '$1')
      // Remove duplicate horizontal rules
      .replace(/(\n---\n)\1+/g, '$1')
      // Fix code block formatting
      .replace(/([^\n])\n*```(\w*)/g, '$1\n\n\n```$2\n') // Three newlines before code block
      .replace(/```(\w*)\n+/g, '```$1\n') // Single newline after opening fence
      .replace(/\n+```\s*/g, '\n```\n\n') // Two newlines after code block
      // Clean up inline code
      .replace(/`([^`]+)`/g, (_, code) => `\`${code.trim()}\``)
      // Ensure proper list indentation
      .replace(/\n\s*[-*+]\s+/g, '\n- ')
      // Add space after list markers
      .replace(/\n-(\S)/g, '\n- $1')
      // Final pass to ensure no more than 2 consecutive newlines
      .replace(/\n{3,}/g, '\n\n')
      .trim() + '\n';

    const blob = new Blob([markdown], { type: 'text/markdown' });
    const url = URL.createObjectURL(blob);

    const filename = `${currentTab.title?.replace(/[^a-z0-9]/gi, '_').toLowerCase() || 'page'}.md`;

    await chrome.downloads.download({
      url: url,
      filename: filename,
      saveAs: true
    });

    status.value = 'Conversion complete!';

    URL.revokeObjectURL(url);
  } catch (error) {
    console.error('Conversion error:', error);
    status.value = `Error: ${error instanceof Error ? error.message : 'Unknown error'}`;
  } finally {
    isConverting.value = false;
  }
}

onMounted(() => {
  if (typeof chrome === 'undefined' || !chrome.tabs) {
    isExtensionContext.value = false;
    status.value = 'This page needs to be loaded as a Chrome extension';
  } else {
    // Get the current tab to check if we can convert it
    chrome.tabs.query({ active: true, currentWindow: true })
      .then(([tab]) => {
        if (!tab?.url || tab.url.startsWith('chrome://')) {
          status.value = 'Cannot convert Chrome system pages';
        }
      })
      .catch(() => {
        status.value = 'Error accessing current tab';
      });
  }
});
</script>

<template>
  <div class="container">
    <h1>Page to Markdown</h1>
    <div class="options">
      <label class="clean-option">
        <input type="checkbox" v-model="removeBeforeHeading">
        <span class="select-none">Remove content before first heading</span>
      </label>
    </div>
    <div class="button-group">
      <button @click="convertToMarkdown(false)"
        :disabled="isConverting || !isExtensionContext || status.includes('Cannot convert')" class="convert-button"
        :title="!isExtensionContext ? 'Please load this as a Chrome extension' : 
                status.includes('Cannot convert') ? 'Cannot convert Chrome system pages' : ''">
        {{ isConverting ? 'Converting...' : 'Download Full Page' }}
      </button>
      <button @click="convertToMarkdown(true)"
        :disabled="isConverting || !isExtensionContext || status.includes('Cannot convert')"
        class="convert-button article-only"
        :title="!isExtensionContext ? 'Please load this as a Chrome extension' : 
                status.includes('Cannot convert') ? 'Cannot convert Chrome system pages' : ''">
        {{ isConverting ? 'Converting...' : 'Download Article Only' }}
      </button>
    </div>
    <p class="status" :class="{
      error: status.includes('Error') || status.includes('Cannot'),
      warning: !isExtensionContext
    }">{{ status }}</p>
  </div>
</template>

<style scoped>
.container {
  width: 300px;
  padding: 16px;
  text-align: center;
  background: #ffffff;
  margin: 0 auto;
}

h1 {
  font-size: 1.5em;
  margin-bottom: 16px;
  color: #2c3e50;
  font-weight: 600;
}

.options {
  background: #f8f9fa;
  border-radius: 4px;
  padding: 8px;
  margin-bottom: 16px;
}

.clean-option {
  display: block;
  margin: 0;
  font-size: 13px;
  color: #444;
  cursor: pointer;
  text-align: left;
  display: flex;
  align-items: center;
}

.clean-option input {
  margin-right: 8px;
  cursor: pointer;
  flex-shrink: 0;
}

.clean-option span {
  line-height: 1.4;
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.convert-button {
  background-color: #42b883;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
  width: 100%;
}

.convert-button.article-only {
  background-color: #4299b8;
}

.convert-button.article-only:hover:not(:disabled) {
  background-color: #3a89a8;
}

.convert-button:hover:not(:disabled) {
  background-color: #3aa876;
}

.convert-button:disabled {
  background-color: #a8a8a8;
  cursor: not-allowed;
  opacity: 0.7;
}

.status {
  margin-top: 16px;
  font-size: 14px;
  color: #666;
  margin-bottom: 0;
}

.status.error {
  color: #dc3545;
}

.status.warning {
  color: #ffc107;
}

button[title]:disabled:hover {
  position: relative;
}

button[title]:disabled:hover::after {
  content: attr(title);
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  padding: 5px 10px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  border-radius: 4px;
  font-size: 12px;
  white-space: nowrap;
  margin-bottom: 5px;
}
</style>
