---
{"dg-publish":true,"permalink":"//2024/8-kindle-my-clippings-txt-obsidian/","tags":["技术研究"],"created":"2024-08-31T17:29:40.365+08:00"}
---

最近在完善自己的生活工作流，其中一项是看完书把笔记存到obsidian，为后续消化整理做准备。

现有的工具试过下面这些，都不能满足我的要求（如实表达，感恩做出的贡献）：
1. 不能在ipad obsidian上运行 (Kindle Highlights)
2. 导入格式挺好看，也给了我启发，但是标记位置什么的都没了 ，不过可以做为备用（Kindle Highlight Importer from HTML)
3. 丢数据 (memo.bookfere.com)
4. 导出太多就收费（clippings.io）
5. 太孤立不能存到obsidian （klib app）

下面的过程完全依靠moonshot kimi助手，即使本人是程序员，但要花时间去研究api也很头疼，于是先靠AI起手，没想到它完成的非常出色，我只微调了一些bug和显示上的不足。

耗时将近4小时，这要是让我自己写不知道要用多长时间。

点击链接查看和 Kimi 智能助手的对话 https://kimi.moonshot.cn/share/cr9eebmgi3pu4dp0437g

下面的总结也是让AI它自己弄的：
> 在这个对话中，我们讨论了如何使用 Obsidian 的 `runjs` 插件来处理 Kindle 的 `My Clippings.txt` 文件，并将其内容转换为 Markdown 格式。以下是对话的主要要点：
> 
> 1. **解析文件内容**：编写了一个 JavaScript 函数 `parseClippings` 来解析 `My Clippings.txt` 文件，提取书籍标题、日期、摘录类型（标注或笔记）、页码位置和摘录文本。
> 
> 2. **组织笔记和标注**：创建了一个函数 `organizeClippings`，用于将笔记（如果它们没有附加到任何标注）组织到标注下。笔记如果没有找到相应的标注，则作为独立的条目保留。
> 
> 3. **写入 Markdown 文件**：编写了一个函数 `writeMarkdown`，将解析和组织后的内容写入新的 Markdown 文件。标注使用引用格式，笔记直接显示在标注内容之下。独立的笔记也按照一定的格式输出。
> 
> 4. **文件选择**：讨论了如何在 iPad 上使用 Obsidian 时选择文件。由于 `runjs` 插件不直接支持弹出选择器，建议将文件预先放置在 Obsidian 的库中，并在脚本中直接指定文件路径。
> 
> 5. **错误处理和日志记录**：在代码中添加了错误处理和日志记录，以帮助调试和跟踪代码执行过程。
> 
> 6. **代码执行**：提供了一个主函数 `main` 来串联整个流程，包括读取文件、解析内容、组织摘录和写入文件。
> 
> 7. **文件路径**：强调了在实际使用中需要将文件路径替换为实际的 `My Clippings.txt` 文件路径。
> 
> 这个对话展示了如何使用 `runjs` 插件来自动化处理和转换 Kindle 摘录的过程，以及如何通过 Obsidian 管理这些信息。

接下来我要做的是把从kindle app，iBook里的笔记，转化成markdown并保存到obsidian中，不过这个不急了，因为暂时用不上，时间全用来做这个反倒没时间看书了。

最终的代码供参考：

```js RunJS_removeme="{n:'Book/Import Kindle Clippings1', t:'s'}"

// 引入Obsidian的API
import * as obsidian from 'obsidian';
const RunJS = this; // 用于runjs插件中的日志记录

// 读取My Clippings.txt文件
async function readMyClippings(filePath) {
  // RunJS.log(`Attempting to read My Clippings from: ${filePath}`);
  try {
    const content = await RunJS.app.vault.adapter.read(filePath);
    // RunJS.log(`Successfully read My Clippings file, length=${content.length}`);
    return content;
  } catch (error) {
    RunJS.log(`Failed to read My Clippings file: ${error}`);
    throw error;
  }
}

// 解析My Clippings.txt文件内容
async function parseClippings(content) {
  // RunJS.log(`Parsing clippings content...`);
  const clippings = content.split(/==========/);
  //RunJS.log(`Number of clippings: ${clippings.length}`);
  const books = {};

  clippings.forEach(clipping => {
    if (clipping.trim() !== '') {
      const lines = clipping.trim().split('\n');
      // RunJS.log(`Number of lines: ${lines.length}`);
      const bookTitle = lines[0].trim();
      // RunJS.log(`Lines[0]: ${bookTitle}`);
      const bookDetails = lines[1].trim();
      // RunJS.log(`Lines[1]: ${bookDetails}`);
      
      const date = bookDetails.match(/添加于 (.*)/)[1];
      const type = bookDetails.includes('的标注') ? 'Highlight' : 'Note';
      const positionMatch = bookDetails.match(/位置 #(\d+)(-(\d+))?/);
      const pageMatch = bookDetails.match(/您在第 (\d+) 页/);//第 19 页
      const page = {
        startPos: positionMatch[1],
        num: pageMatch[1]
      };
      if (positionMatch[3]) {
        page.endPos = positionMatch[3];
      }
      //const page = bookDetails.match(/位置 #(\d+)/) ? `Page: ${bookDetails.match(/位置 #(\d+)/)[1]}` : '';
      const text = lines.slice(3).join('\n').trim();
      //RunJS.log(`lines.slice(3).join('\n').trim(): ${text}`);
      const clippingInfo = {
        title: bookTitle,
        date: date,
        type: type,
        page: page,
        text: text,
        notes: [] // 初始化笔记数组
      };

      if (!books[bookTitle]) {
        books[bookTitle] = [];
        //RunJS.log(`Found new book: ${bookTitle}`);
      }

      books[bookTitle].push(clippingInfo);
      //RunJS.log(`Parsed clipping for: ${bookTitle}`);
    }
  });

  //RunJS.log(`Finished parsing clippings.`);
  return books;
}

function organizeClippings(books) {
  for (const [bookTitle, clippings] of Object.entries(books)) {
    // 先存储所有标注的引用
    const highlights = clippings.filter(clipping => clipping.type === 'Highlight');
    // 再处理笔记，尝试找到对应的标注
    for (let i = 0; i < clippings.length; i++) {
      const note = clippings[i];
      if (note.type === 'Note') {
        let attached = false;
        for (const highlight of highlights) {
          if (note.page.startPos >= highlight.page.startPos && (!highlight.page.endPos || note.page.startPos <= highlight.page.endPos)) {
            highlight.notes = highlight.notes || [];
            highlight.notes.push(note);
            attached = true;
            break;
          }
        }
        if (!attached) {
          // 如果笔记没有附加到任何标注，保留为独立笔记
          note.standAlone = true;
        }
      }
    }
  }
}

// 将解析后的内容写入新的Markdown文件
async function writeMarkdown(books, targetFolder) {
  //RunJS.log(`Writing clippings to Markdown files...`);
  for (const [bookTitle, clippings] of Object.entries(books)) {
    //const markdownContent = clippings.join('\n\n');
    const markdownContent = [];
    for (const clipping of clippings) {
      if (clipping.type === 'Highlight') {
        // 处理标注
        markdownContent.push(`### 标注，第${clipping.page.num}页，位置  ${clipping.page.startPos}${clipping.page.endPos ? `-${clipping.page.endPos}\n` : '\n'}`);
        markdownContent.push(`\n> ${clipping.text}\n\n`);
        // 处理附加到标注的笔记
        if (clipping.notes && clipping.notes.length > 0) {
        markdownContent.push(`> [!笔记]\n`);
        for (const note of clipping.notes) {
            markdownContent.push(`> ${note.text}\n`);
          }
        }
        markdownContent.push('\n---\n');
      } else if (clipping.type === 'Note' && clipping.standAlone) {
        // 处理独立笔记
        markdownContent.push(`### 笔记，第${clipping.page.num}页，位置 ${clipping.page.startPos}\n`);
        markdownContent.push(`\n>[!${clipping.text}]\n\n---\n`);
      }
    }

    const markdownPlainContent = markdownContent.join('');
    const targetPath = `${targetFolder}/${bookTitle.replace(/\s+/g, '_')}.md`;
    try {
	  let file = RunJS.app.vault.getAbstractFileByPath(targetPath);
	  if(file === null) {
		  //RunJS.log("File not exists, create");
		  await RunJS.app.vault.create(targetPath, markdownPlainContent);
	  } else {
		  //RunJS.log("File exists, modify");
		  await RunJS.app.vault.modify(file, markdownPlainContent);
	  }
      
      //RunJS.log(`Successfully created Markdown file: ${targetPath}`);
    } catch (error) {
      RunJS.log(`Error creating Markdown file for ${bookTitle}: ${error}`);
    }
  }
  //RunJS.log(`Finished writing Markdown files.`);
}

async function main()
{ // 立即执行的异步函数
  try {
    //RunJS.log("Start");
	const myClippingsPath = '1 📥收集箱/📚读书心得/Kindle/My Clippings.txt'; // 替换为你的My Clippings.txt文件路径
	const targetFolder = '1 📥收集箱/📚读书心得/Kindle'; // 替换为你想要创建Markdown文件的目标文件夹路径

    //RunJS.log(`Starting the process...`);
    
    // 读取My Clippings.txt文件
    const content = await readMyClippings(myClippingsPath);
    //RunJS.log(`Successfully read My Clippings file.`);
    
    // 解析My Clippings.txt文件内容
    const books = await parseClippings(content);
    organizeClippings(books); // 新增组织摘录的步骤
    // 将解析后的内容写入新的Markdown文件
    await writeMarkdown(books, targetFolder);
    //RunJS.log(`Process completed successfully.`);
  } catch (error) {
    RunJS.log(`An error occurred during the process: ${error}`);
  }
};

main().then().catch(RunJS.log);
```


<div class="transclusion internal-embed is-loaded"><a class="markdown-embed-link" href="///" aria-label="Open link"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="svg-icon lucide-link"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg></a><div class="markdown-embed">




*本站所有内容版权及解释权归本站内容发布者（夏至夕阳）所有，如需转载请注明出处并附上本站链接，如有侵权请联系。*


</div></div>
