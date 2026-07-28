---
description: Use when the user wants to audit, write, or apply SEO titles and meta descriptions across posts on a Substack publication, especially when working in Claude Cowork with browser automation available. Enumerates every published post via Substack's public archive API, flags which ones are missing custom SEO, reads each post's actual text before writing anything, applies the SEO fields live through the post editor, and verifies the result via the API. Also useful for a one-off "does my Substack have SEO gaps?" audit even if the user doesn't want changes applied yet.
---

# Substack Post SEO (audit, write, apply, verify)

This skill turns "improve my Substack SEO" into a repeatable four-phase workflow: discover, research, draft, apply, verify. It was built from direct experience applying SEO to 43 live posts across two Substack publications inside Cowork, including the dead ends that didn't work.

## Prerequisites

- The user must be logged into Substack as the post's author (or an editor with publish access) in the connected browser. This skill relies on browser automation (Claude-in-Chrome tools in Cowork) - navigate, javascript_tool, and ideally get_page_text and find.
- Works on both *.substack.com subdomains and custom domains (e.g. example.com) - the API paths below are relative to whatever domain the publication resolves to.
- If browser tools aren't available in the session, fall back to producing a worksheet (see Phase 2) for the user to apply manually - Settings -> SEO Options exists in every Substack post editor regardless of automation.

## Phase 1 - Discover: enumerate every post and find SEO gaps

Substack exposes a public JSON archive endpoint that includes each post's current SEO state. Fetch it in pages (each post object is large, so ~25 per page is a safe batch size to avoid truncating tool output):

GET https://{domain}/api/v1/archive?sort=new&limit=25&offset=0

Increment offset by 25 until a page returns fewer than 25 items. Each post object includes:
- id - the numeric post ID, used later to open the editor directly
- slug, canonical_url - for reading the live post and mapping worksheet entries back to URLs
- title, subtitle - the on-page title/subtitle (NOT what you should use for SEO copy - see Phase 2)
- search_engine_title, search_engine_description - null if no custom SEO has been set yet. This is the gap signal.

Run this via javascript_tool on any page already on the domain (navigate to the homepage first) rather than one-page-at-a-time browser reads - it's much cheaper on context:

```js
let all = [];
for (let off = 0; off < 500; off += 25) {
  const r = await fetch(`/api/v1/archive?sort=new&limit=25&offset=${off}`);
    const j = await r.json();
      if (!j.length) break;
        all = all.concat(j.map(p => ({id: p.id, slug: p.slug, st: !!p.search_engine_title, sd: !!p.search_engine_description})));
          if (j.length < 25) break;
          }
          JSON.stringify(all);
          ```

          Report the total post count and how many already have st/sd both true vs. how many are gaps. Do not trust a prior worksheet's post count - always re-enumerate. A publication may have more posts than any earlier audit covered (this is exactly the mistake that prompted writing this skill: an "all posts" pass had actually only covered 31 of 43 published posts).

          ## Phase 2 - Research: read the real text before writing anything

          This is the step that is easiest to skip and most damaging to skip. Do not write SEO copy from title/subtitle alone - read the actual post body first. Titles and subtitles are often stylized or oblique (Substack authors frequently use a hook-y title that doesn't describe the content), and skipping this step produces SEO copy that misdescribes the post.

          For each post needing SEO:
          1. Navigate to its canonical_url (or https://{domain}/p/{slug}).
          2. Use get_page_text to extract the full article text.
          3. If get_page_text / navigate hits a rate limit or the browser is unavailable, WebFetch on the canonical URL is a fallback - but Substack rate-limits bursts of WebFetch calls (HTTP 429) faster than it rate-limits browser navigation. If reading many posts in a row, prefer the browser, or pace WebFetch calls with delays, or delegate the reading to a subagent that can retry with backoff.
          4. For a large publication, delegate batches of "read this post's full text and summarize the concrete, checkable claims in it" to subagents to parallelize without blowing the context budget of the main session - but do the actual SEO drafting yourself once you have the grounded summaries, since voice and judgment about what's genuinely searchable shouldn't be delegated blindly.

          ## Phase 3 - Draft: write grounded, constrained SEO copy

          For each post, produce:
          - SEO title: 60 characters or fewer. Should describe what the post is actually about, not just restate the post's stylized title. Can differ from the on-page title.
          - SEO description: aim for 140-160 characters (Substack's own guidance is 50-160). One or two sentences, plain and specific, summarizing the real argument or finding - not a teaser, not keyword stuffing.
          - Match the publication's actual voice. Don't impose your own stylistic tics (e.g. if you tend to add em dashes and the author doesn't use them, don't add them here either - ask or check a few of the author's own posts if unsure).
          - Flag, rather than silently invent, anything you can't verify from the text (a claimed statistic, a named product, a person's title) - this matters more for SEO copy than it seems, because meta descriptions are exactly the kind of text that gets quoted back at the author in search results.

          Before applying anything, present the drafted title and description per post as a worksheet (a markdown table or list) and get the user's go-ahead, unless they've already pre-approved "apply directly" for this batch. Note any posts you're intentionally skipping (e.g. very low-search-demand content like weekly roundups) rather than silently dropping them - say so explicitly.

          ## Phase 4 - Apply: set the fields live via the editor

          Two ways in, both land on the same UI:

          - Direct editor URL (fast, use this when you have the post ID): https://{domain}/publish/post/{id}
          - Via the live post: open the post, then the "..." menu near the like/share icons, then Edit.

          Once in the editor, the SEO fields live inside the Post settings modal, in a collapsed SEO Options section, reachable via the gear-icon Settings button (bottom-right of the editor). This modal is somewhat flaky to drive with plain clicks - the Settings button sometimes needs a moment after page load before it responds, and clicking it twice in quick succession toggles it back shut. Do not try to write these fields directly via Substack's internal /api/v1/drafts/{id} PUT endpoint - a full-object PUT with only the SEO fields changed was rejected (HTTP 400) in testing, likely because the endpoint expects a specific field subset per request; the safe path is through the UI.

          The most reliable approach found in practice is a single deterministic javascript_tool call per post that: checks whether the settings modal is already open before clicking Settings (avoids the double-click toggle bug), expands SEO Options if the fields aren't visible yet, sets both fields using the native property setter plus input/change events (required because this is a React-controlled input - just setting .value without dispatching events won't register), and clicks Save:

          ```js
          const sleep=ms=>new Promise(r=>setTimeout(r,ms));
          const title="YOUR SEO TITLE HERE";
          const desc="YOUR SEO DESCRIPTION HERE";
          const isOpen=()=>[...document.querySelectorAll('*')].some(e=>e.textContent==='Post settings'&&!e.children.length);
          if(!isOpen()){const s=[...document.querySelectorAll('button')].find(b=>b.textContent.trim()==='Settings');if(s)s.click();}
          await sleep(1000);
          if(!document.querySelector('input[placeholder="Enter a custom title..."]')){
            const lbl=[...document.querySelectorAll('*')].find(e=>e.textContent==='SEO Options'&&!e.children.length);
              const t=lbl&&(lbl.closest('button')||lbl.parentElement);
                if(t)t.click();
                }
                await sleep(700);
                const ti=document.querySelector('input[placeholder="Enter a custom title..."]');
                const de=document.querySelector('textarea[placeholder="Enter a custom description..."]');
                if(!ti||!de){ 'NO_FIELDS'; } else {
                  function setVal(el,val){
                      const proto=el.tagName==='TEXTAREA'?HTMLTextAreaElement.prototype:HTMLInputElement.prototype;
                          const setter=Object.getOwnPropertyDescriptor(proto,'value').set;
                              setter.call(el,val);
                                  el.dispatchEvent(new Event('input',{bubbles:true}));
                                      el.dispatchEvent(new Event('change',{bubbles:true}));
                                        }
                                          setVal(ti,title); setVal(de,desc);
                                            await sleep(500);
                                              const saveBtn=[...document.querySelectorAll('button')].find(b=>b.textContent.trim()==='Save'&&!b.disabled);
                                                if(saveBtn){ saveBtn.click(); await sleep(800); }
                                                }
                                                ```

                                                navigate to each post's editor URL, wait roughly 5-6 seconds for the editor to fully hydrate (navigating and immediately running the script before the editor has loaded is the single most common cause of "Settings click did nothing"), then run the script above with that post's title/description substituted in. This changes nothing else on the post - body, title, slug, publish state are untouched; the SEO fields autosave with no republish step required.

                                                ## Phase 5 - Verify

                                                Never trust a "clicked Save" result alone - confirm the write actually persisted, using the same read-only API endpoint from Phase 1 (safe, no side effects):

                                                ```js
                                                async function chk(id){
                                                  const r = await fetch(`/api/v1/posts/by-id/${id}`);
                                                    const j = await r.json();
                                                      const p = j.post || j;
                                                        return {slug: p.slug, ok: !!p.search_engine_title && !!p.search_engine_description,
                                                                  tlen: (p.search_engine_title||'').length, dlen: (p.search_engine_description||'').length};
                                                                  }
                                                                  // run for every post id just touched, then also re-check the FULL set from Phase 1
                                                                  // to catch any that silently failed to save
                                                                  ```

                                                                  Check every post touched this run, and re-run the full-publication check from Phase 1 at the end so a save that silently failed (it happens - the editor occasionally didn't register a save on the first try in testing) doesn't slip through unnoticed. Flag any post where tlen > 60 or dlen > 160 too.

                                                                  ## Summary checklist for the agent running this skill

                                                                  1. Enumerate every post via the archive API - don't rely on a prior count.
                                                                  2. Read each post's actual text before drafting copy.
                                                                  3. Present a worksheet and get sign-off before applying, unless told to apply directly.
                                                                  4. Apply via the deterministic JS routine above, one post at a time, with load-wait before each.
                                                                  5. Verify every post via the read-only posts/by-id endpoint, and re-check full coverage at the end.
                                                                  6. Report exact numbers: X/Y posts covered, any that failed, any intentionally skipped.
                                                                  
