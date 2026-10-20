# Vulkanstorm chat Markdown: carry-over spec for vkstorm-aleph

Source tree: `C:\Dev\vkstorm-release-ref` at `08c07b4cba` (origin/vkstorm-release). Firestorm base: `upstream/master` = `0de5043af0`. All paths below are relative to that tree. Line numbers are from the release checkout.

Scope note up front: the brief anticipated a wider dialect (underline, strikethrough, code spans, code blocks, labelled links inside the parser). None of that exists. The Vulkanstorm chat dialect is exactly three things: `**strong**`, `_emphasis_` (normal mode) or `_` as an italic toggle (emote mode), and `__` as an escaped literal underscore, plus two protection mechanisms (caller-supplied literal byte ranges for URLs, and built-in emoticon-mouth detection). Labelled links `[url label]` are a separate, pre-existing Firestorm URL-registry feature that the parser only protects; Vulkanstorm's only release change there is a settings default. The whole feature landed in one commit, `020ea7193e` "Preserve Markdown chat formatting and message-local fixes" (16 files, +1585/-31), plus `7d5d138b42` "Enable labelled chat hyperlinks by default" (one settings line).

---

## 1. Grammar (normative spec)

### 1.1 Entry point and modes

`LLMarkdown::parseEmphasis(text: UTF-8 bytes, emote: bool, literal_ranges: [(start,end))) -> Vec<Span>` (`indra/llui/llmarkdown.h:95-96`, `indra/llui/llmarkdown.cpp:134`).

- `emote = false`: **normal chat mode**. `_..._` is emphasis (italic) with CommonMark-like flanking; `**...**` is strong (bold); `__` is an escaped literal underscore.
- `emote = true`: **emote mode** (`/me` lines, `CHAT_STYLE_IRC`, and the input-side synonyms `:` (MU* pose, gated by `AllowMUpose`) and a leading `_` (always on), which `FSCommon::applyMuPose` rewrites to `/me ` before sending: `indra/newview/fscommon.cpp:146-168`). The base text is italic; each single `_` toggles italic off/on; `**...**` still applies; `__` is a literal underscore.
- `literal_ranges`: half-open byte ranges the caller wants left untouched (in practice URL matches; see 2.3). Bytes inside any protected range can never be a delimiter or part of an escape.
- Empty input yields zero spans (`llmarkdown.cpp:137-140`).
- Invariant: concatenating the span texts reconstructs the input exactly, except that each escaped `__` collapses to `_` (`llmarkdown.h:78-80`, tests enforce via `ensure_roundtrip`).

The parser is byte-oriented over UTF-8. It never decodes code points except when locating emoticon eyes (1.3). Character classes, all ASCII-only unless stated (`llmarkdown.cpp:39-42`):

- `word byte` = ASCII `[0-9A-Za-z]` or `_` **or any byte >= 0x80** (every non-ASCII character, including non-Latin letters, CJK punctuation, NBSP, zero-width space and emoji, counts as a word character).
- `whitespace` = ASCII `isspace` (space, `\t`, `\n`, `\v`, `\f`, `\r`) for flanking; the strong-content emptiness check uses only `" \t\r\n"` (`llmarkdown.cpp:206`).
- Byte at an out-of-range position reads as `\0` (`llmarkdown.cpp:44-47`).

### 1.2 Processing order

The order is load-bearing; it is what produces the nesting and protection rules.

1. Copy caller `literal_ranges` into `protected`.
2. **Emoticon protection** (both modes): scan left to right; every underscore run that qualifies as an emoticon mouth (1.3) is appended to `protected`. (`llmarkdown.cpp:157-165`)
3. **Escape detection** (normal mode only): scan left to right; at each `__` whose two bytes are both unprotected, record an escape at that offset, add `[offset, offset+2)` to `protected`, and skip the pair. Consecutive pairs each count (`____` = two escapes = renders `__`). Note the scan is greedy-left: `___` = escape at 0 plus a free `_` at 2. (`llmarkdown.cpp:174-186`)
4. **Pass 1, strong** (both modes). (1.4)
5. Emote mode: **toggle pass** then return. (1.6)
6. Normal mode: **Pass 2, emphasis**. (1.5)
7. **Emission** of spans (intersection of strong partition with emphasis matches). (1.7)
8. Normal mode: **escape collapsing**, deleting the second byte of every recorded escape from whichever span contains it. (`llmarkdown.cpp:400-415`)

### 1.3 Emoticon (kaomoji) mouth protection

Spec source: `llmarkdown.cpp:49-105` plus `doc/markdown_emoticons.md`. Purpose: `^_^` must not lose its underscore in normal chat (where a lone `_` that can open formats to end of message) and `o_o` must not toggle in emotes.

An underscore run starting at byte `pos` is an emoticon mouth iff all of the following hold (`emoticonMouthEnd`, `llmarkdown.cpp:66-105`):

1. `text[pos] == '_'`, `pos > 0`, and `text[pos-1] != '_'` (the run is examined from its first underscore only).
2. `end` = index of the first non-underscore byte after `pos`; the run must **not** extend to end of string.
3. Left eye = the whole UTF-8 character immediately before `pos` (walk back over continuation bytes `10xxxxxx`); right eye = the whole UTF-8 character starting at `end`.
4. Both eyes are in the eye vocabulary:
   - ASCII eyes (single byte): `^ ~ o O 0 1 2 3 4 5 6 7 8 9 > < - T t U u = + x X ; : @ * . ? Q q P p Y y E e N n ' ` / \` (`llmarkdown.cpp:53`).
   - Unicode eyes (multi-byte, exact character membership): U+0CA0 ಠ, U+0CA5 ಥ, U+00B0 °, U+30FB ・, U+2022 •, U+00AC ¬, U+30FC ー, U+4E00 一, U+FFE3 ￣, U+FF20 ＠, U+FF3E ＾, U+2299 ⊙, U+2609 ☉, U+25CE ◎, U+2565 ╥, U+00F2 ò, U+00F3 ó, U+00F5 õ, U+00F9 ù, U+00D4 Ô, U+00D7 ×, U+FFE2 ￢, U+2192 →, U+2190 ←, U+21BC ↼, U+21C0 ⇀, U+25A1 □, U+03BC μ, U+4E2A 个, U+30CE ノ, U+FF89 ﾉ, U+30FD ヽ, U+FF3C ＼, U+FF1E ＞, U+FF1C ＜, U+FF0B ＋, U+02D8 ˘, U+B208 눈, U+25E3 ◣, U+25E2 ◢, U+2606 ☆ (41 characters; bytes at `llmarkdown.cpp:54-61`, decoded above). Implementation detail: the C++ does a substring search of the eye's bytes in the concatenated vocabulary string; because the eye is always a complete character, this is equivalent to set membership.
5. **Word-embedding guard** ("letter/digit eyes must not be embedded in larger words; punctuation eyes may touch surrounding text", `doc/markdown_emoticons.md:10-11`): let `is_letter_or_digit(b)` = word byte and not `_`. Reject if (`text[left]` is letter/digit AND `left > 0` AND `text[left-1]` is letter/digit) OR (`text[end]` is letter/digit AND `byteAt(right)` is letter/digit). Because non-ASCII bytes are word bytes, every Unicode eye is treated as a "letter" eye: it is rejected when the preceding or following character is any letter, digit or non-ASCII character. ASCII punctuation eyes (`^ ~ > < - = + ; : @ * . ? ' `` / \`) may touch words (`hello^_^there` stays literal, test 45).
6. The protected range is the entire underscore run `[pos, end)`, so extended mouths `^____^` are protected as a unit; scanning resumes at `end`.

Consequences: `photo_order` and `123_456` are not faces; `o_o` inside a word is not a face; a face's underscores are plain text in both modes and keep whatever style surrounds them (`_^_^_` is italic `^_^`, `**^_~**` is bold `^_~`). Faces are detected before escape detection, so `^__^` is a face, not an escape. Spaced mouths (`^ _ ^`), combining accents on eyes and shrug arms (`¯\_(ツ)_/¯`) are explicitly out of scope (`doc/markdown_emoticons.md:33-36`). Worked example for the shrug in normal mode: neither `_` is a face (`(` and `)` are not eyes); the first `_` can open (before `\` is a non-word byte, after `(` is not whitespace) and the second `_` can close (before `)` is not whitespace, after `/` is a non-word byte), so the reference renders `¯\(ツ)/¯` with `(ツ)` italic; in emote mode both underscores are toggles. The Rust port may add `(`/`)` to the eye vocabulary if that is unwanted, but the reference does not, and no test covers it.

### 1.4 Strong `**...**` (both modes)

`llmarkdown.cpp:188-214`.

- Opener: `**` at `pos` with both bytes unprotected. **No flanking rule**: intraword `see**this**` works; `**` preceded by anything opens.
- Closer: the next `**` at or after `pos+2` whose two bytes are both unprotected (protected `**`, for example inside a URL, are skipped over and the search continues).
- **Unclosed opener**: if no closer exists and the byte at `pos+2` exists and is not whitespace, the strong span runs to end of input with no closing delimiter span emitted. `** tail` and a bare `**` at end are plain.
- Content must be non-empty and contain at least one byte outside `" \t\r\n"` (`** **` is plain; `** a**` is bold `" a"`). No whitespace rule on the closer side (`**a **` is bold `"a "`).
- After a match, scanning resumes after the closer; strong spans do not nest in strong spans.
- Strong spans may contain `_` freely: underscores inside a strong span are **never** emphasis delimiters in normal mode (`inside_strong` check, `llmarkdown.cpp:216-227, 297, 309`). So `**bold _x_ end**` renders the underscores literally. Only bold-inside-italic nests, never italic-inside-bold. (Escapes `__` inside bold still collapse in normal mode: `**a__b**` -> `a_b`, test 43.)
- A single `*` is never a delimiter (`2*3*4`, `*_*`, censoring).

### 1.5 Emphasis `_..._` (normal mode only)

`llmarkdown.cpp:107-131, 293-326`.

- Opener `_` at `pos`: not protected, not inside any strong span (delimiters included), and `canOpenEmphasis`: byte after exists and is not whitespace; byte before is absent (start of text) or is **not** a word byte. Because `_` and all non-ASCII bytes are word bytes, `some_var_name`, `café_au_lait`, `abc_def_ ghi` and `«_x_»` cannot open.
- Closer: scan forward from `pos+1` for the first `_` that is not inside a strong span, not protected, and `canCloseEmphasis`: byte before is not whitespace (and pos>0); byte after is absent or not a word byte. Candidates failing the test are skipped and become content (`_a_b_ c` -> italic `a_b`). The scan runs to end of input (newlines do not stop it: `_first\nsecond` is italic to the end, test 38).
- **Unclosed opener**: if no closer is found, the emphasis span extends to end of input with no closing delimiter, and scanning stops (there can be nothing after it). `a _lonely underscore` -> `a ` + italic `lonely underscore`. This is the "message-local" rule: formatting never leaks into the next message because each message is parsed independently (`llmarkdown.h:45`, test 38 "next message resets").
- `_ nope _` is plain (opener followed by whitespace). A bare `_` is plain.
- Emphasis content may contain whole strong spans (bold-in-italic). An emphasis opener cannot sit inside a strong span, and a strong span cannot be partially overlapped because strong is resolved first and underscores inside it are ignored.

### 1.6 Emote toggle mode (`emote = true`)

`llmarkdown.cpp:229-291`. Base state: italic ON (the whole emote is italic; the callers set the base font style to ITALIC unconditionally, see section 3).

Walk the text left to right with state `italic_on = true` and a region start:

- When reaching the start of a strong match (from 1.4): flush the current region as `EMOTE_TOGGLE_ON` (if italic_on) or `EMOTE_TOGGLE_OFF` (if not), then emit `STRONG_DELIM "**"`, `STRONG content`, and (only if the match was closed) `STRONG_DELIM "**"`; continue after it. The toggle state is unchanged by a strong span; underscores inside a strong span are not toggles and are emitted raw (no `__` collapsing inside bold in emote mode, unlike normal mode).
- `_` not protected:
  - if the next byte is `_` and not protected: flush, emit `EMOTE_LITERAL "_"` (two bytes consumed, one `_` rendered, no toggle);
  - else: flush, emit `EMOTE_DELIM "_"`, flip `italic_on`.
  `___` therefore = literal `_` then a toggle (`action___tail` -> `OLeF`).
- Protected underscores (faces, URLs) are ordinary text in the current region.
- At end of input flush the final region. A trailing `_` with nothing after it is simply consumed (`...pants_` -> `OeFeOe`, test 22).
- Toggle state is per call; the callers pass each message separately so "the toggle never survives the line" (`llmarkdown.h:93-94`). Note the parser itself does **not** reset at `\n` inside one call.

### 1.7 Span emission and classification (normal mode)

`llmarkdown.cpp:328-398`.

Strong matches partition the input. Gaps between strong spans are emitted with base type `PLAIN`; strong content is emitted with base type `STRONG`; strong delimiters as `STRONG_DELIM` (closing one only if present). Within each region, every emphasis match that intersects it is applied: the `_` delimiter bytes become `EMPHASIS_DELIM`; the emphasis content bytes become `EMPHASIS`, **even when they lie inside a strong span's content**. That is why `_some **bold** text_` -> `d I D I D I d` and `_**strong**_` -> `d D I D d`: the bold text inside italic is classified `EMPHASIS`, not `STRONG`. The header comment says "the caller merges both" (`llmarkdown_test.cpp:239-241`) but neither consumer does (section 2.2): in the current viewer, bold nested in italic renders italic only (delimiters are hidden correctly). Treat this as a known quirk; the Rust port should either reproduce the span types verbatim for test parity and merge at the style layer (recommended: keep a `bold` flag on the span, see section 5), or decide deliberately to drop bold.

### 1.8 What is explicitly not supported

- `*italic*`, `***`, `~~strike~~`, `__bold__` (double underscore is an escape), underline, code spans/blocks, headings, lists, blockquotes, links of any Markdown form (`[text](url)` is plain text; wiki-style `[url label]` is handled by the URL registry, not Markdown), HTML, backslash escapes.
- Italic nested inside bold; strong nested in strong; overlapping spans.
- Unicode-aware flanking (all non-ASCII is "word"); Unicode whitespace (NBSP, ideographic space are word bytes, not whitespace).
- Any per-line reset inside a single parse call.

### 1.9 Delimiter stripping helper

`LLMarkdown::stripEmphasisDelimiters(text, emote)` (`llmarkdown.cpp:422-445`) runs `parseEmphasis` with **no** literal ranges and drops `EMPHASIS_DELIM`/`STRONG_DELIM`/`EMOTE_DELIM`, keeping everything else (so `EMOTE_LITERAL` yields `_`). Fast path: if the text has no `_` and no `**` it returns unchanged. It is covered by tests 30-37 and 41-47 but is not called by any production code in the release tree (the console now carries per-character styles instead; `indra/newview/fsconsoleutils.cpp:37` only includes the header). Port it anyway: it is the cheapest "visible text" oracle for tests and for plain-text sinks (logs, notifications, accessibility).

---

## 2. Output model and how the UI consumes it

### 2.1 Span list

```
enum ESpanType : u8 {
  PLAIN,            // ordinary text, keeps incoming style
  EMPHASIS,         // text inside _..._ -> italic
  STRONG,           // text inside **...** -> bold
  EMPHASIS_DELIM,   // matched '_' (hidden)
  STRONG_DELIM,     // matched '**' (hidden)
  EMOTE_TOGGLE_OFF, // emote: region with base italic toggled off (spoken part)
  EMOTE_TOGGLE_ON,  // emote: region with base italic on (action)
  EMOTE_DELIM,      // emote: the '_' toggle byte (hidden)
  EMOTE_LITERAL,    // emote: escaped '__' rendered as "_"
}
struct Span { type: ESpanType, text: String }   // llmarkdown.h:56-73
```

The parser carries **no** colour, font, link or label information: spans are purely text plus a semantic type. There are no nested spans; nesting is flattened into adjacent spans with the inner type (1.7). Style attributes are decided by the consumer from the span type plus the message's base style.

### 2.2 Consumption in LLTextBase (chat history, IM history, nearby-chat toasts)

`LLTextBase::appendText` (`indra/llui/lltextbase.cpp:2766-2824`):

1. Gate: `mParseMarkdown` (widget flag, `lltextbase.h:450, 821`; XUI param `parse_markdown`, default false, `lltextbase.cpp:199`) AND the incoming `LLStyle::Params` is not a link segment (`!input_params.is_link`) AND the text contains `_` or `**`. Otherwise the text is appended unparsed.
2. `message_params` = widget default style overwritten by the caller's params. `base_flags` = font style bits parsed from `message_params.font.style` (`NORMAL|BOLD|ITALIC|UNDERLINE`). `emote_flags` starts equal to `base_flags`.
3. Literal ranges: if `mParseHTML`, repeatedly run `LLUrlRegistry::findUrl` over the remaining text and record `[start, end+1)` **byte** ranges of each URL match (`lltextbase.cpp:2781-2792`). This is the "URL protection".
4. `parseEmphasis(new_text, message_params.markdown_emote, literal_ranges)`; `markdown_emote` is a transient `LLStyle::Params` hint (`indra/llui/llstyle.h:58-63`, default false `llstyle.cpp:52`) that is not stored on `LLStyle`.
5. Per span, starting from `flags = emote ? emote_flags : base_flags`:
   - `EMPHASIS_DELIM`, `STRONG_DELIM`: skipped (hidden).
   - `EMOTE_DELIM`: `emote_flags ^= ITALIC`, skipped.
   - `EMPHASIS`: `flags |= ITALIC`.
   - `STRONG`: `flags |= BOLD` (in emote mode this is on top of the current toggle state, so bold inside the spoken part is bold-upright, inside the action bold-italic; test 29).
   - `EMOTE_TOGGLE_ON`: `flags |= ITALIC; emote_flags = flags`. `EMOTE_TOGGLE_OFF`: `flags &= ~ITALIC; emote_flags = flags`.
   - `PLAIN`, `EMOTE_LITERAL`: unchanged.
   Then `appendTextImpl(span.text, params-with-font.style(flags))`. Base BOLD (shout) or ITALIC (whisper) from the message style is preserved and combined.
6. `appendTextImpl` then does the normal URL pass on each span (`lltextbase.cpp:2638-2760`): URL matches inside a span become link segments with label `match.getLabel()`, `link_href = match.getUrl()` (`indra/llui/llurlmatch.cpp:62`), tooltip `match.getTooltip()` (`lltextbase.cpp:2698-2704`), the grey query part, and optional trust icon. Because every URL match is a protected range, a URL always lies wholly inside one span and keeps the span's bold/italic flags. Caveat: a URL regex can swallow a trailing `**` (`\S+`), so `**see https://x/y** tail` becomes an unclosed bold with the `**` visibly inside the URL (test 40; expected behaviour).
7. Spans become `LLNormalTextSegment`s holding an `LLStyle` (colour, readonly colour, alpha, font name/size/style, `link_href`, `is_link`, tooltip). Newlines inside a span are turned into line-break segments by `appendAndHighlightText` (`lltextbase.cpp:3058-3082`); the style continues across them.

### 2.3 Consumption in LLConsole (on-screen nearby chat / IM console overlay)

`indra/llui/llconsole.cpp:476-502, 550-614, 672-731, 836-860`.

- `addConsoleLine(text, color, session, styleflags, markdown_offset, markdown_emote)`: `markdown_offset` is the length in **UTF-32 code units** of the un-parsed prefix (sender name + delimiter); `-1` disables parsing for that line (`llconsole.h:172-173`). Each queued line carries `(offset, emote)` in `mMarkdownMessages`.
- `Paragraph` construction first rewrites wiki-labelled links in the paragraph text to their labels (Firestorm code, `llconsole.cpp:851-874`), then `makeParagraphColorSegments`: the body after the offset is converted to UTF-8, parsed with literal ranges = URL matches in the body plus every occurrence of each known URL label (`llconsole.cpp:561-582`), and the same flag logic as 2.2 produces a `U8` style per **wchar** (`mCharacterStyles`, `llconsole.h:139`), base style = the `styleflags` passed in (ITALIC for emote/whisper, BOLD for shout). Delimiter spans are dropped from `mParagraphText` so the stored text is the visible text (the original is kept in `mSourceText`).
- Line wrapping walks style runs, selects a per-run `LLFontGL` with that style (`llconsole.cpp:672-731`) and renders each `LineColorSegment` with its own font (`llconsole.cpp:377-390`). Colour is uniform per paragraph. The console has no mouse handling: links are not clickable there (`grep handleMouseDown llconsole.cpp` is empty).

### 2.4 Links: labels, targets, display of the target, scheme restrictions

Not part of the Markdown parser; part of `LLUrlRegistry`/`LLUrlEntry` (Firestorm/Linden code, unchanged by Vulkanstorm except a settings default).

- Labelled (wiki-style) links: `[http://host/path Label]` (`LLUrlEntryHTTPLabel`, regex `\[(https?|ftp)://\S+[ \t]+[^\]]+\]`, `indra/llui/llurlentry.cpp:344-369`) and `[secondlife:///app/... Label]` / `[hop://... Label]` (`LLUrlEntrySLLabel`, `llurlentry.cpp:1567-1611`). Only `http`, `https`, `ftp`, `secondlife`, `hop` schemes can be labelled. Markdown `[text](url)` is not recognised anywhere.
- Label vs target: segment text is the label; `link_href` is the URL (`getUrlFromWikiLink`, `llurlentry.cpp:130-140`); the label is returned by `getLabelFromWikiLink` (`llurlentry.cpp:114-128`).
- Anti-deception guards (all pre-existing Firestorm/LL):
  1. If the label itself parses as a URL, the label is discarded and the real URL is shown instead (`llurlentry.cpp:355-359, 1575-1579`).
  2. `isWikiLinkCorrect` (`llurlentry.cpp:185-240`) strips U+200B and folds confusable dots (U+2024, U+FE52, U+FF0E, U+05C5), colons (U+02D0, U+FF1A, U+2236, U+FE55), fullwidth solidus and Cyrillic shha before re-testing whether the label looks like a URL (also prefixing `https://` to labels containing `.com`/`www.`); if it does, the labelled form is rejected entirely (`indra/llui/llurlregistry.cpp:277-283`).
  3. Tooltip: for HTTP/FTP labelled links the hover tooltip is the **real URL** (`llurlentry.cpp:361-364`). For `secondlife://`/`hop://` labelled links the tooltip is the generic type tooltip ("Click to run the secondlife:// command", `strings.xml:351`), **not** the target; that is a gap the Rust UI should close (show the resolved target on hover).
  4. User setting `FSDisableLabeledChatLinks` (`settings.xml:19660-19670`): when true, labelled links from untrusted content (chat, IM, group notices, object text) are not collapsed to a label; the raw URL is shown and linked (`llurlregistry.cpp:256-264`; preference text `panel_preferences_firestorm.xml:319-327`). Vulkanstorm release default: **0** (labels enabled) per `7d5d138b42` "Enable labelled chat hyperlinks by default". Current Firestorm upstream defaults it to 1 and additionally has `FSDisableLabeledChatLinksNearbyChat` plus an "instant large hint" hover preview of the destination; neither the split key nor the hint preview is in the Vulkanstorm release tree (the pr141/pr144 "labelled link preview" work exists only on canary/devel; `git merge-base --is-ancestor` confirms `62c13d4f04`, `98840a93b4`, `768e9a25cf`, `fd501b84df`, `fa94f11a3e` are not ancestors of release).
- Built-in viewer text is "trusted content" and bypasses the setting.

---

## 3. Where it is applied, and what is excluded

All wiring is from `020ea7193e`; the Firestorm base has none of it.

| Surface | Code | Parse? | Emote detection | Notes |
|---|---|---|---|---|
| Local chat history, IM, group IM, ad-hoc (FSChatHistory, the live chat widget) | `indra/newview/fschathistory.cpp:1298-1302` (`mParseMarkdown = true` in ctor), `1546-1557` (emote), `1949-1956` (body append) | Yes, in both plain-text and rich layouts | `FSCommon::is_irc_me_prefix` (`/me ` or `/me'`, `fscommon.cpp:85-89`) or `CHAT_STYLE_IRC` | Emotes are italic **unconditionally** (the `EmotesUseItalic` gate was removed in the emote branch; it still exists for the moderator-style path at `fschathistory.cpp:1618`). In rich mode the sender's name SLURL is prepended to the body before parsing (`fschathistory.cpp:1917-1934`); it is a URL match so it is protected. Name, timestamp and delimiter segments are appended in separate calls; names are `is_link` and `<nolink>` wrapped, so they do not parse. |
| Nearby chat toast panels | `indra/newview/llchatitemscontainerctrl.cpp:190-206, 325-340` (`LLFloaterIMNearbyChatToastPanel::addMessage/init`) | Yes, `setParseMarkdown(true)` around the body append, off otherwise (`:247`) | `chat_style == CHAT_STYLE_IRC` | Base style ITALIC for emotes (unconditional), BOLD shout, ITALIC whisper. |
| Nearby chat console overlay (v1-style) | `indra/newview/fsconsoleutils.cpp:98-102, 125-126, 171-175` | Yes; the body after `name + delimiter` | `is_irc_me_prefix` or `CHAT_STYLE_IRC` | Includes agent, object and "other" (system/radar) sources; the "other" branch parses in normal mode with offset `name + " "` or 0. Gated by `FSUseNearbyChatConsole` (default 1). |
| IM / group IM echoed to console | `fsconsoleutils.cpp:234-262` | Yes; body after `"IM: " + name + delimiter` | `/me` prefix | Gated by `FSLogImToChatConsole` / `FSLogGroupImToChatConsole` (both default 0). |
| LLChatHistory (Linden widget) | `indra/newview/llchathistory.cpp:1181, 1298, 1376, 1634-1636` | Wired, but the whole file is `#if 0` (`llchathistory.cpp:29, 1660`) | | Dead code; FSChatHistory replaces it. |
| Chat input line / editors | none | No | | `parse_markdown` defaults false; the raw text the user typed is kept. |
| IM toasts (LLToastIMPanel), notification toasts, group notices, script dialogs, inspectors, floater titles, chat logs on disk | none | No | | Only the four surfaces above set `parse_markdown`/`setParseMarkdown`. Logs receive the raw message with delimiters. |
| Outgoing text | `fscommon.cpp:160-165` | n/a | | Leading `_` is rewritten to `/me ` on send (always on). MU* `:` rewrite gated by `AllowMUpose` (default 0). |

"Message-local fixes" in the commit title refers to the behaviours that make formatting state live and die with one message: unmatched openers format only to the end of their own message (`llmarkdown.h:45`, test 38), emote toggle state is per message, the console tracks per-message `(offset, emote)` so formatting never bleeds into the prefix or the next line, and emote italics became unconditional so the toggle has a defined base.

Settings and defaults (release `settings.xml`):

| Key | Default | Effect |
|---|---|---|
| `FSDisableLabeledChatLinks` | 0 (`settings.xml:19669`; upstream has 1) | 1 = show raw URL instead of label for untrusted labelled links |
| `EmotesUseItalic` | 1 (`:5181-5190`) | No longer consulted for normal emote italics (unconditional); only in the moderator-style branch |
| `FSEmphasizeShoutWhisper` | 1 (`:24278`) | Shout BOLD / whisper ITALIC base style in FSChatHistory; combined with markdown flags |
| `AllowMUpose` | 0 (`:976`) | `:pose` -> `/me pose` on send |
| `FSUseNearbyChatConsole` | 1 (`:664`) | Console surface active |
| `FSLogImToChatConsole`, `FSLogGroupImToChatConsole` | 0 (`:676, 687`) | IM to console |
| `PlainTextChatHistory` | 1 (`:9900`) | Layout only; Markdown parses in both modes |

There is **no** master on/off switch for chat Markdown in the release tree (no `ChatMarkdown*` key exists); it is always on for the surfaces above.

---

## 4. Test corpus (47 tests, `indra/llui/tests/llmarkdown_test.cpp`)

Signature legend (`llmarkdown_test.cpp:56-80`): `P`=PLAIN, `I`=EMPHASIS, `B`=STRONG, `d`=EMPHASIS_DELIM, `D`=STRONG_DELIM, `O`=EMOTE_TOGGLE_ON, `F`=EMOTE_TOGGLE_OFF, `e`=EMOTE_DELIM, `L`=EMOTE_LITERAL. `parse(x)` = normal mode plus roundtrip assertion (concatenation equals input). `strip(x, emote)` = `stripEmphasisDelimiters`. Indices are span indices. Strings are shown as Rust string literals.

| # | Mode | Input | Expected |
|---|---|---|---|
| 1 | normal | `"hello world"` | 1 span, sig `P`, text `"hello world"` |
| 2 | normal | `""` | 0 spans |
| 3 | normal | `"_italic_"` | `dId`; [1]=`"italic"` |
| 4 | normal | `"**bold**"` | `DBD`; [1]=`"bold"` |
| 5 | normal | `"say _wow_ now"` | `PdIdP`; [0]=`"say "`, [2]=`"wow"`, [4]=`" now"` |
| 6 | normal | `"some_var_name"` | `P`; text unchanged |
| 7 | normal | `"abc_def_ ghi"` | `P` |
| 8 | normal | `"a _lonely underscore"` | `PdI`; last=`"lonely underscore"` |
| 9 | normal | `"a **lonely star"` | `PDB`; last=`"lonely star"` |
| 10 | normal | `"2*3*4"` | `P` |
| 11 | normal | `"** **"` | `P` |
| 12 | normal | `"**two words**"` | `DBD`; [1]=`"two words"` |
| 13 | normal | `"_a_ and **b**"` | `dIdPDBD`; [1]=`"a"`, [5]=`"b"` |
| 14 | normal | `"_**strong**_"` | `dDIDd`; [2]=`"strong"` |
| 15 | normal | `"_a_b_ c"` | `dIdP`; [1]=`"a_b"` |
| 16 | normal | `"caf\u{e9}_au_lait"` | `P`; text unchanged |
| 17 | normal | `"_caf\u{e9}_"` | `dId` |
| 18 | normal | `"that was _great_!"` | `PdIdP`; [2]=`"great"`, [4]=`"!"` |
| 19 | normal | `"_ nope _"` | `P` |
| 20 | normal | `"**bold** _dangling"` | `DBDPdI`; [1]=`"bold"`, last=`"dangling"` |
| 21 | normal | `"_some **bold** text_"` | `dIDIDId`; [1]=`"some "`, [3]=`"bold"`, [5]=`" text"` |
| 22 | emote | `"throws a glass at Edward_ \"You idiot!\" _pants_"` | `OeFeOe`; [0]=`"throws a glass at Edward"`, [2]=`" \"You idiot!\" "`, [4]=`"pants"`, [5]=`"_"` |
| 23 | emote | `"nods_ yes _grins"` | `OeFeO`; [2]=`" yes "`, [4]=`"grins"` |
| 24 | emote | `"nods_ yes, that's right, I almost got you killed _he grins"` | `OeFeO`; [2]=`" yes, that's right, I almost got you killed "`, [4]=`"he grins"` |
| 25 | emote | `"waves_ hi _and_ bye _smiles"` | `OeFeOeFeO`; [2]=`" hi "`, [4]=`"and"`, [6]=`" bye "`, [8]=`"smiles"` |
| 26 | normal | `"nods_ yes, that's right _he grins"` | roundtrip; `PdI` |
| 27 | emote | `"shrugs_ whatever"` | `OeF`; [0]=`"shrugs"`, [2]=`" whatever"` |
| 28 | emote | `"sighs__ loudly_ ok"` | `OLOeF`; [0]=`"sighs"`, [1]=`"_"`, [2]=`" loudly"`, [3]=`"_"`, [4]=`" ok"` (roundtrip intentionally lossy) |
| 29 | emote | `"stares_ **seriously**? _blinks"` | `OeFDBDFeO`; [0]=`"stares"`, [2]=`" "`, [4]=`"seriously"`, [6]=`"? "`, [8]=`"blinks"` |
| 30 | strip emote | `"nuzzles the nape of your neck_ yes Mistress"` | `"nuzzles the nape of your neck yes Mistress"` |
| 31 | strip emote | `"shrugs_ whatever"` | `"shrugs whatever"` |
| 32 | strip emote | `"waves_ hi _smiles"` | `"waves hi smiles"` |
| 33 | strip emote | `"uses__literal"` | `"uses_literal"` |
| 34 | strip normal | `"some_var_name"` | `"some_var_name"` |
| 35 | strip normal | `"it _is_ perfect"`; `"perfect **now**"` | `"it is perfect"`; `"perfect now"` |
| 36 | strip normal | `"plain chat line"` | unchanged (fast path) |
| 37 | normal | `"besides... you _love_ us precisely because we **are** trouble, boo _giggles"` | `PdIdPDBDPdI`; last=`"giggles"`; strip normal = `"besides... you love us precisely because we are trouble, boo giggles"` |
| 38 | normal | `"_first\nsecond"`; `"**first\nsecond"`; `"plain next message"`; `"**"`; `"_"`; `"** tail"` | `dI`; `DB`; `P`; `P`; `P`; `P` |
| 39 | emote | `"waves_ **hello"` | roundtrip; `OeFDB` |
| 40 | normal, literal `[(6,32)]` | `"**see https://example.com/_path** tail"` | roundtrip; `DB`; last=`"see https://example.com/_path** tail"` |
| 41 | emote | `" says something outrageous!_  \"You all are insane!\""` | roundtrip; `OeF`; [0]=`" says something outrageous!"`, last=`"  \"You all are insane!\""`; strip emote = `" says something outrageous!  \"You all are insane!\""` |
| 42 | emote | `" adds to the puffy fat cat cheeks!_ =<^^>__"` | `OeFL`; last=`"_"`; strip emote = `" adds to the puffy fat cat cheeks! =<^^>_"`. Also emote `"action__tail"`->`OLO`; emote `"action_**quote**"`->`OeDBD`; emote `"action___tail"`->`OLeF`; emote `"next action"`->`O` |
| 43 | strip normal | `"a__b"`->`"a_b"`; `"__"`->`"_"`; `"____"`->`"__"`; `"_a__b_"`->`"a_b"`; `"**a__b**"`->`"a_b"`; `"_a__b"`->`"a_b"`. Normal with literal `[(0,len)]`: `"https://example.com/a__b"` | roundtrip; `P` |
| 44 | both | for face in `^_^ o_o 0_0 ^_~ o_O O_o T_T u_u -_- >_< @_@ *_*`: input `"hello (" + face + ")!"` | normal `P`; emote roundtrip and `O`; strip normal and strip emote both return the input unchanged |
| 45 | mixed | `parse("_^_^_")`->`dId`; strip normal `"_o_o_"`->`"o_o"`; strip normal `"**^_~**"`->`"^_~"`; `parse("hello^_^there")`->`P`; emote `"photo_order"`->`OeF`; emote `"smiles ^_^_ hi o_o _waves ^_~"` | roundtrip; `OeFeO`; [2]=`" hi o_o "`; strip emote = `"smiles ^_^ hi o_o waves ^_~"` |
| 46 | both | each face in the corpus below, as the whole input | normal `P`, emote `O`, roundtrip, strip in both modes returns the face unchanged |
| 47 | mixed | `"^_^ o_o 0_0 ^_~ Q_Q ;_; ._. ^____^"`: normal `P`, emote `O`; strip normal `"_^____^_"`->`"^____^"`; strip normal `"word__tail"`->`"word_tail"`; emote `"a_b"`->`OeF`; emote `"123_456"`->`OeF`; normal with literal `[(0,len)]` `"https://example.com/^__^/_path"` | roundtrip; `P` |

Test 46 corpus (`llmarkdown_test.cpp:601-654`), ASCII then Unicode (escapes are Rust `\u{}`):

```
Q_Q  ;_;  :_;  ;_:  '_'  /_;  ._.  <_<  >_>  ?_?  +_+  X_X  x_X  =_=  ~_~  ^_-  p_-  -_q  e_e  n_n  Y_Y
^_^;  (^_^)/  (-_-)zzz  ((d[-_-]b))  (??_??)  (--_--)  ^__^  ^_________^  o___O
\u{CA0}_\u{CA0}   \u{CA0}__\u{CA0}   \u{CA5}_\u{CA5}   \u{B0}_\u{B0}   \u{30FB}_\u{30FB}   \u{AC}_\u{AC}
\u{30FC}_\u{30FC}   \u{FFE3}_\u{FFE3}   \u{FF20}_\u{FF20}   \u{2299}_\u{2299}   \u{2299}_\u{2609}   \u{2565}_\u{2565}
\u{F2}_\u{F3}   \u{F3}_\u{F2}   \u{F5}_o   \u{F9}_u   o_\u{D4}   \u{D7}_\u{D7}   \u{FFE2}_\u{FFE2}
\u{2192}_\u{2192}   \u{2190}_\u{2190}   \u{25A1}_\u{25A1}
(\u{30CE}_<\u{3002})   (\u{B208}_\u{B208})   (\u{2606}_@)
```

Port note for test 46: `(\u{30CE}_<\u{3002})` works because the right eye is ASCII `<`; U+3002 after it is irrelevant. The word-embedding guard matters for `\u{F5}_o` (left eye õ preceded by nothing, right eye `o` followed by end) and `o_\u{D4}`.

Suggested Rust harness: a `sig(&[Span]) -> String` helper with the same letters, a `parse()` wrapper asserting roundtrip, and one `#[test]` per row; keep the numbering in test names (`t14_strong_in_emphasis`) so failures map back to this table.

---

## 5. Recommendation for the Rust port

**Port the custom parser 1:1. Do not use pulldown-cmark.** Reasons: the dialect is intentionally tiny and diverges from CommonMark in ways that matter to SL users (single `*` never formats; `__` is an escape, not strong; `_` is a toggle in emotes; unclosed openers format to end of message; emoticon protection; byte-level flanking; italic cannot nest in bold). Restricting pulldown-cmark would require fighting its delimiter-run algorithm, inline-code/link/autolink/HTML handling, and its Unicode flanking (which would break tests 16, 19, 44-47), and you would still need the emote pass and the escape semantics on top. The C++ is 446 lines including licence and comments; the logic is roughly 250 lines. Expect **250-350 lines of Rust** for `parse_emphasis` + `strip_delimiters` + eye vocabulary, and **~400 lines of tests** for the 47 cases. No dependencies needed.

Design for the Rust side:

- Operate on `&[u8]` of the UTF-8 string, exactly like the C++, and build span text with `String::from_utf8_unchecked`/`from_utf8(...).unwrap()` on slices whose boundaries are guaranteed to be char boundaries (all cut points are at ASCII delimiters, at `pos`/`end` of underscore runs, or at literal-range edges supplied by the caller; validate caller ranges against `is_char_boundary` and clamp/skip bad ones rather than panic, since the C++ `end > remaining.size()` guard at `lltextbase.cpp:2789` shows they come from a regex engine). Keep `is_word_byte(b) = b.is_ascii_alphanumeric() || b == b'_' || b >= 0x80` verbatim: the tests depend on non-ASCII-is-word (16, 17) and on ASCII-only whitespace.
- Represent the result as `Vec<Span { kind, text }>` with the nine kinds, and additionally expose a resolved form `Vec<StyledRun { text, italic: bool, bold: bool, hidden: bool }>` produced by a function that implements the flag logic of `lltextbase.cpp:2793-2822` (base flags in, emote toggle state machine). That resolver is the place to fix the bold-in-italic quirk (1.7): track "inside strong content" during emission so a run can be both bold and italic, while keeping the span kinds verbatim for the ported tests. Decide explicitly and document it; the two reference consumers render bold-in-italic as italic only.
- Keep `literal_ranges` as an input and have the chat pipeline feed URL match byte ranges (and, for any console-like surface that substitutes labels before parsing, the label occurrences) exactly as `lltextbase.cpp:2781-2792` and `llconsole.cpp:561-582` do. Run URL detection first, then Markdown, then linkify within each run, so a URL never straddles a style boundary.
- Emoticon eyes: store the 41 Unicode eyes as a `&[char]` (or a sorted `[u32]` for binary search) plus the ASCII byte set; detect eyes by decoding one `char` before `pos` and one at `end` (`str::chars().next_back()` on the prefix slice and `chars().next()` on the suffix slice). Reproduce the word-embedding guard at the byte level (it inspects raw bytes `text[left-1]`, `text[left]`, `text[end]`, `text[right]`), otherwise the Unicode corpus rows and `photo_order`/`123_456` diverge.
- Unicode/RTL: the parser needs no grapheme segmentation; eyes are single scalar values, never grapheme clusters (combining accents are out of scope by spec). RTL text is unaffected by parsing (delimiters are ASCII), but the renderer must apply bold/italic per run after bidi reordering; Hebrew/Arabic words adjacent to `_` are "word bytes", so `_עברית_` opens (start of text) and closes (end), while `א_ב` stays literal, which matches user expectation. NBSP/U+3000/U+200B are word bytes, not whitespace: `_a_<NBSP>` will not close and italicises to end of message. If you later want Unicode-aware flanking, add it behind a flag and keep the byte-mode tests green.
- Surfaces: apply to the same set (chat/IM/group history bodies, nearby-chat toasts, console-style overlay, IM echo) with the same emote detection (`/me ` or `/me'` prefix, IRC style) and the same base styles (emote ITALIC unconditional; shout BOLD / whisper ITALIC when the equivalent of `FSEmphasizeShoutWhisper` is on). Keep input editors, logs, notifications and names/timestamps unparsed. Preserve the input-side rewrite of a leading `_` to `/me `.
- Links: carry label and target as separate fields on the link run; always show the real target on hover for every labelled scheme (closing the `secondlife://`/`hop://` tooltip gap), reject labels that look like URLs after confusable folding, restrict labelled schemes to `http`, `https`, `ftp`, `secondlife`, `hop`, and expose a "show raw URL instead of label for untrusted content" preference defaulting to labels-on to match the Vulkanstorm release default (`FSDisableLabeledChatLinks = 0`).
