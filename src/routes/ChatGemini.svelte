<script lang="ts">
    import bot_logo from '$lib/images/bot.svg';
    import user_logo from '$lib/images/user.svg';
    import {
        PUBLIC_PERPLEXITY_API,
        PUBLIC_GEMINI_API,
    } from '$env/static/public';
    import DOMPurify from 'isomorphic-dompurify';
    import 'highlight.js/styles/github-dark.css';
    import { onMount } from 'svelte';
    import {
        GoogleGenAI,
        createPartFromUri,
        type Part,
        type ContentListUnion,
    } from '@google/genai';

    import { markedHighlight } from 'marked-highlight';
    import hljs from 'highlight.js';
    import { Marked } from 'marked';

    type PerplexityCitation = string;

    type PerplexityChoice = {
        index: number;
        finish_reason: string;
        message: {
            role: string;
            content: string;
        };
        delta?: {
            role?: string;
            content?: string;
        };
    };

    type PerplexityResponse = {
        id: string;
        model: string;
        created: number;
        usage: {
            prompt_tokens: number;
            completion_tokens: number;
            total_tokens: number;
            search_context_size: string;
        };
        citations: PerplexityCitation[];
        object: string;
        choices: PerplexityChoice[];
    };

    let preview_image: string | null = $state('');
    let file_upload: Blob | string | null = $state(null);
    let search: HTMLInputElement | null = null; // Checkbox, if check -> use perplexity service
    let message = $state(''); // Message input
    /* let text_response = ''; // Response from LLM */
    let chat_container: HTMLDivElement | null = null; // Use to scroll bottom for each DOM updated
    let chat_history: ChatMessage[] = $state([]);
    let loading = $state(false);
    let text_area_el: HTMLTextAreaElement;
    let error_msg: string | unknown = $state('');
    const ai = new GoogleGenAI({ vertexai: false, apiKey: PUBLIC_GEMINI_API });
    let loading_text_part = $state('Vui lòng chờ...');

    let system_instruction = `You are a lively, witty, and genuinely helpful assistant. Make complex problems easy to understand by breaking them into simple, logical steps. Use relatable analogies and real-life comparisons whenever they help clarify things. Keep the tone engaging, friendly, and playful—just enough to make learning fun without distraction. You aim to make the user feel like they're chatting with a clever friend who always has their back. Always reflect the language of the user. Use icons in your answer to make it vivid.
You are given a reference answer from Perplexity. Use it to help answer the user's question, but improve or clarify as needed.`;

    $effect(() => {
        if (text_area_el && message === '') {
            text_area_el.style.height = 'auto';
            text_area_el.style.height = text_area_el.scrollHeight + 'px';
            /* (async () => { */
            /*     await tick(); */
            /*     if (text_area_el && message === '') { */
            /*         text_area_el.style.height = 'auto'; */
            /*         text_area_el.style.height = text_area_el.scrollHeight + 'px'; */
            /*     } */
            /* }) */
        }
    });

    const chat = ai.chats.create({
        model: 'gemini-2.5-flash-preview-05-20',
        config: {
            systemInstruction: system_instruction,
        },
    });

    async function handle_pasting(e: ClipboardEvent) {
        loading = true;
        const data = e.clipboardData;
        let items;
        if (data) {
            items = data.items;
        } else {
            return;
        }

        for (let i = 0; i < items.length; i++) {
            const item = items[i];
            if (item.type.startsWith('image/')) {
                e.preventDefault();
                const file = item.getAsFile();
                file_upload = file;

                if (file) {
                    try {
                        /* let base64; */
                        const reader = new FileReader();

                        // The loadend event of the FileReader interface is fired when a file read has completed,
                        // successfully or not. (https://developer.mozilla.org/docs/Web/API/FileReader/loadend_event )
                        reader.onloadend = () => {
                            preview_image = reader.result as string;
                            /* console.log(base64); */
                        };
                        reader.onerror = (error) => {
                            console.error(
                                'Error reading file as base64:',
                                error
                            );
                            error_msg = 'Failed to read image file for API.';
                            preview_image = null;
                        };
                        // https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsDataURL
                        // read this `file`. When the read operation is finished
                        //      the readyState property becomes DONE, and the loadend event is triggered
                        reader.readAsDataURL(file);
                    } catch (e) {
                        alert(e);
                    } finally {
                        loading = false;
                        /* file_upload = null; */
                        /* preview_image = null; */
                    }
                }
            } else {
                loading = false;
            }
        }
    }

    function auto_resize(node: HTMLTextAreaElement) {
        const resize = () => {
            node.style.height = 'auto';
            node.style.height = node.scrollHeight + 'px';
        };

        node.addEventListener('input', resize);

        requestAnimationFrame(resize);

        return {
            destroy() {
                node.removeEventListener('input', resize);
            },
        };
    }

    /* let render_response: string | Promise<string> = ''; */

    function formatDate(date: Date) {
        const h = '0' + date.getHours();
        const m = '0' + date.getMinutes();
        return `${h.slice(-2)}:${m.slice(-2)}`;
    }

    const marked = new Marked(
        markedHighlight({
            highlight(code, lang) {
                if (lang && hljs.getLanguage(lang)) {
                    return hljs.highlight(code, { language: lang }).value;
                }
                return hljs.highlightAuto(code).value;
            },
        })
    );

    async function search_engine(
        prompt: string
    ): Promise<PerplexityResponse | undefined> {
        let userMessageContent: any;
        if (preview_image) {
            userMessageContent = [
                { type: 'text', text: prompt },
                {
                    type: 'image_url',
                    image_url: {
                        url: preview_image,
                    },
                },
            ];
        } else {
            userMessageContent = prompt;
        }

        const bodyData = {
            model: 'sonar-pro', // Use "sonar-pro" as per Perplexity's multimodal example
            stream: false,
            messages: [
                { role: 'system', content: 'Be precise and concise.' },
                { role: 'user', content: userMessageContent },
            ],
        };
        const options = {
            method: 'POST',
            headers: {
                Authorization: `Bearer ${PUBLIC_PERPLEXITY_API}`,
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(bodyData),
        };

        try {
            const response = await fetch(
                'https://api.perplexity.ai/chat/completions',
                options
            );
            if (!response.ok) throw new Error(await response.text());
            return await response.json();
        } catch (err) {
            error_msg = err;
            return undefined;
        }
    }

    function scrollToBottom() {
        if (chat_container) {
            chat_container.scrollTo({
                top: chat_container.scrollHeight,
                behavior: 'smooth',
            });
        }
    }

    type ChatMessage = {
        sender: 'user' | 'bot';
        content?: string;
        htmlContent?: string | any;
        name: string;
        img: string | null;
        time: string;
    };

    function escapeHTML(str: string | null) {
        if (str === null) return '';
        return str
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    async function handle_file_upload_change(event: Event) {
        const input = event.target as HTMLInputElement;
        if (input.files && input.files.length > 0) {
            const file = input.files[0]; // store blob
            file_upload = file;

            if (file.type.startsWith('image/')) {
                /* preview_image = URL.createObjectURL(file_upload); */
                const reader = new FileReader();

                // The loadend event of the FileReader interface is fired when a file read has completed,
                // successfully or not. (https://developer.mozilla.org/docs/Web/API/FileReader/loadend_event )
                reader.onloadend = () => {
                    preview_image = reader.result as string;
                };
                reader.onerror = (error) => {
                    console.error('Error reading file as base64:', error);
                    error_msg = 'Failed to read image file for API.';
                    preview_image = null;
                };
                // https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsDataURL
                // read this `file`. When the read operation is finished
                //      the readyState property becomes DONE, and the loadend event is triggered
                reader.readAsDataURL(file);
            } else {
                preview_image = null;
            }
        }
    }

    async function process_file_upload() {
        let content: ContentListUnion | null = [];
        if (file_upload) {
            const file = await ai.files.upload({
                file: file_upload,
                config: {
                    displayName: (file_upload as any).name,
                },
            });
            loading_text_part = 'Đang tải file lên';

            let get_file = await ai.files.get({ name: file.name as string });

            while (get_file.state === 'PROCESSING') {
                loading_text_part = 'Đang xử lý file';

                get_file = await ai.files.get({ name: file.name as string });
                /* console.log(`current file status: ${get_file.state}`); */
                /* console.log('File is still processing, retrying in 5 seconds'); */

                await new Promise((resolve) => {
                    setTimeout(resolve, 5000);
                });
            }

            if (file.state === 'FAILED') {
                loading = false;
                loading_text_part = 'Vui lòng chờ...';
                throw new Error('File processing failed.');
            }

            if (file.uri && file.mimeType) {
                const fileContent = createPartFromUri(file.uri, file.mimeType);
                content.push(fileContent);
                loading_text_part = 'Đã xử lý xong';
            }
        } // end file upload processing
        return content;
    }

    async function use_perplexity_search_engine(msg: string) {
        let combined_prompt = '';
        let context_from_perplexity = '';

        if (search?.checked && msg) {
            loading_text_part = 'Đang tìm kiếm thông tin...';
            let response_from_perplexity: PerplexityResponse | undefined =
                await search_engine(msg);
            /* console.log(response_from_perplexity); */
            if (response_from_perplexity?.choices[0].message.content) {
                context_from_perplexity = context_from_perplexity.concat(
                    response_from_perplexity?.choices[0].message.content
                );

                context_from_perplexity =
                    context_from_perplexity.concat('\nCitations: \n');

                if (response_from_perplexity.citations.length > 0) {
                    response_from_perplexity.citations.map((citation, i) => {
                        context_from_perplexity =
                            context_from_perplexity.concat(
                                `\t${i + 1}: ${citation}\n`
                            );
                    });
                }
            }
            combined_prompt = `Perplexity says: "${context_from_perplexity}"\n\nUser asks: ${msg}`;
        }
        return combined_prompt;
    }

    async function generate(msg: string | null) {
        // 1. if the user upload file -> process it
        // 2. if the user use perplexity search engine -> use it
        // 3. call gemini

        // if i dont assign any file, then i will use combined_prompt

        /* if (loading || (msg?.trim() === '' && !file_upload)) return; */

        loading = true;
        message = ''; // reset message (prompt)

        /* let marked_parsed: any = marked.parse( */
        /*     escapeHTML(msg ? msg : `\n${(file_upload as any).name}`) */
        /* ); */

        let m: any = marked.parse(escapeHTML(msg) +
                                           (file_upload ?  `<br/><i>Tệp đính kèm: ${(file_upload as any).name}</i>` : ''));
        chat_history = [
            ...chat_history,
            {
                sender: 'user',
                htmlContent: DOMPurify.sanitize(m),
                name: 'YOU',
                img: user_logo,
                time: formatDate(new Date()),
            },
        ];

        let content: ContentListUnion | null = [];
        // process file upload
        if (file_upload) {
            try {
                console.log('waiting');
                content = await process_file_upload();
            } catch (e) {
                console.log(e);
                chat_history.pop(); // pop user message
                return;
            }
        }
        // use perplexity if check
        let combined_prompt = '';
        if (search?.checked && msg) {
            combined_prompt = await use_perplexity_search_engine(msg);
        }

        // combined between combined_prompt and content into content -> WHAT?
        /* if (content.length > 0 && combined_prompt !== '') { */
        /*     content.push(combined_prompt); */
        /* } else if (content.length > 0 && msg && msg.trim() !== '') { */
        /*     content.push(msg); */
        /* } */

        loading_text_part = 'Vui lòng chờ...';

        scrollToBottom();
        /* await tick(); */

        const bot_msg: ChatMessage = {
            sender: 'bot',
            content: '',
            htmlContent: '',
            name: 'BOT',
            img: bot_logo,
            time: formatDate(new Date()),
        };
        chat_history.push(bot_msg);

        let text_response = '';

        try {
            let response_stream;
            if (content && content.length > 0) {
                let content_send: Part[] = [
                    {
                        fileData: {
                            fileUri: (content[0] as any).fileData.fileUri,
                            mimeType: (content[0] as any).fileData.mimeType,
                        },
                    },
                    {
                        text: combined_prompt
                            ? combined_prompt
                            : msg
                              ? msg
                              : '',
                    },
                ];

                /* response_stream = await ai.models.generateContentStream({ */
                /*     config: { */
                /*         systemInstruction: system_instruction, */
                /*     }, */
                /*     model: 'gemini-2.5-flash-preview-05-20', */
                /*     contents: content, // included `combined_prompt` */
                /* }); */

                response_stream = await chat.sendMessageStream({
                    message: content_send,
                });
            } else {
                // if it's not contain media type
                response_stream = await chat.sendMessageStream({
                    message:
                        combined_prompt !== ''
                            ? combined_prompt
                            : msg
                              ? msg
                              : '',
                });

                loading = false;
                loading_text_part = 'Vui lòng chờ...'; // reset this text to default
                /* throw new Error('Invalid prompt.'); */
            }

            for await (const chunk of response_stream) {
                text_response += chunk.text;
                const botIndex = chat_history.length - 1;

                chat_history[botIndex] = {
                    ...chat_history[botIndex],
                    htmlContent: marked.parse(text_response),
                };

                /* scrollToBottom(); */
            }
        } catch (e) {
            error_msg = e;
            chat_history.pop();
        } finally {
            file_upload = null;
            preview_image = null;
            loading = false;
        }
    }

    function handleEnter(e: KeyboardEvent) {
        if (e.key === 'Enter' && !e.shiftKey) {
            if (!loading) {
                e.preventDefault();
                generate(message);
            }
        }
    }
    // Init welcome message
    onMount(async () => {
        loading = true;
        const response: any = await chat.sendMessage({ message: 'Hi' });

        chat_history = [
            {
                sender: 'bot',
                htmlContent: response.text,
                name: 'BOT',
                img: bot_logo,
                time: formatDate(new Date()),
            },
        ];

        loading = false;
    });
</script>

<section class="msger">
    {#if error_msg !== ''}
        {alert(error_msg)}
    {/if}

    <header class="msger-header">
        <div class="msger-header-title">
            <i class="fas fa-comment-alt"></i> Chat chit và tìm kiếm thông tin,
            made with 🔥 by and
            <a
                href="https://github.com/ducchinhpro123/just-a-simple-chat-bot.git"
                >open source</a
            >
        </div>
        <div class="msger-header-options">
            <span>⚙️ </span>
        </div>
    </header>

    <div class="msger-chat" bind:this={chat_container}>
        {#each chat_history as c}
            <div class="msg {c.sender === 'bot' ? 'left-msg' : 'right-msg'}">
                <img
                    alt={c.sender === 'bot' ? 'bot' : 'user'}
                    src={c.sender === 'bot' ? bot_logo : user_logo}
                    class="msg-img"
                />

                <div class="msg-bubble">
                    <div class="msg-info">
                        <div class="msg-info-name">{c.name}</div>
                        <div class="msg-info-time">{c.time}</div>
                    </div>

                    <div class="msg-text">
                        {@html c.htmlContent}
                        <!--
                       -{#if c.sender === 'bot' && c.htmlContent}
                       -    {@html c.htmlContent }
                       -{:else}
                       -    {c.content}
                       -{/if}
                       -->
                    </div>
                </div>
            </div>
        {/each}

        {#if loading}
            <div class="typing-indicator">
                <div class="dot"></div>
                <div class="dot"></div>
                <div class="dot"></div>
            </div>
        {/if}
    </div>

    <div class="msger-inputarea">
        <textarea
            bind:value={message}
            bind:this={text_area_el}
            onpaste={handle_pasting}
            onkeydown={handleEnter}
            use:auto_resize
            class="msger-input"
            placeholder="Nhập tin nhắn..."
            rows="1"
        ></textarea>

        <div class="controls-group">
            <label class="search-checkbox">
                <input type="checkbox" bind:this={search} /> Sử dụng Perplexity 🧠
            </label>

            <div class="action-buttons-group">
                <button
                    onclick={() => generate(message)}
                    type="submit"
                    class="msger-send-btn"
                    disabled={loading ||
                        (message.trim() === '' && file_upload == null)}
                >
                    {#if loading}
                        {loading_text_part}
                    {:else}
                        Gửi
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            width="24"
                            height="24"
                            viewBox="0 0 24 24"
                        >
                            <path
                                fill="currentColor"
                                d="M4 18.5v-5.154L9.846 12L4 10.654V5.5L19.423 12z"
                            />
                        </svg>
                    {/if}
                </button>

                <label for="file-upload-input" class="file-upload-label">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="24"
                        height="24"
                        viewBox="0 0 24 24"
                        class="attach-icon"
                    >
                        <path
                            fill="currentColor"
                            d="M16.5 6v11.5c0 2.21-1.79 4-4 4s-4-1.79-4-4V5a2.5 2.5 0 0 1 5 0v10.5c0 .55-.45 1-1 1s-1-.45-1-1V6H10v9.5a2.5 2.5 0 0 0 5 0V5c0-2.21-1.79-4-4-4S7 2.79 7 5v12.5c0 3.04 2.46 5.5 5.5 5.5s5.5-2.46 5.5-5.5V6h-1.5Z"
                        />
                    </svg>
                </label>
                <input
                    type="file"
                    id="file-upload-input"
                    onchange={handle_file_upload_change}
                    disabled={loading}
                    class="visually-hidden-file-input"
                />
            </div>
        </div>
    </div>

    {#if file_upload}
        <p class="file-upload-display">{(file_upload as any).name}</p>
    {/if}

    {#if preview_image}
        <img
            src={preview_image}
            alt="Uploaded file preview"
            class="uploaded-file-preview-image"
        />
    {/if}
</section>

<style>
    :global(html, body) {
        height: 100vh;
        margin: 0;
        padding: 0;
        overflow: hidden; /* Prevent body scrolling */
    }

    :global(body) {
        display: flex;
        justify-content: center;
        align-items: center;
        background-image: var(--body-bg);
        font-family: Helvetica, sans-serif;
    }

    .file-upload-display {
        margin: 10px;
    }

    .uploaded-file-preview-image {
        display: block; /* Allows margin auto for horizontal centering */
        max-width: 250px; /* Adjust as needed */
        max-height: 200px; /* Adjust as needed */
        width: auto; /* Maintain aspect ratio */
        height: auto; /* Maintain aspect ratio */
        margin: 10px auto 15px auto; /* top / horizontal auto / bottom / horizontal auto - for spacing and centering */
        border: 2px solid #e0e0e0; /* A light border */
        border-radius: 4px; /* Slightly rounded corners */
        object-fit: contain; /* Ensures the whole image is visible within the dimensions, doesn't crop */
        background-color: #f9f9f9; /* A light background for transparent images or while loading */
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); /* Subtle shadow */
    }
    /*
    ---------
     */
    /* === DESKTOP FIRST STYLES === */
    .msger-inputarea {
        display: flex;
        align-items: flex-start;
        padding: 10px;
        border: 1px solid #ccc;
        gap: 15px;
    }

    .msger-input {
        flex-grow: 1;
        border: 1px solid #ccc;
        padding: 10px;
        min-height: 30px;
        resize: none;
        line-height: 1.3;
        max-height: 70px;
    }

    .msger-input::placeholder {
        opacity: 1;
        font-style: normal;
        font-size: 1.5em;
    }

    .controls-group {
        display: flex;
        flex-direction: column;
        align-items: flex-start;
        gap: 10px;
        padding-top: 5px;
        flex-shrink: 0;
    }

    .search-checkbox {
        display: flex;
        align-items: center;
        cursor: pointer;
        font-size: 0.8em;
    }

    .search-checkbox input[type='checkbox'] {
        appearance: none;
        -webkit-appearance: none;
        -moz-appearance: none;
        width: 18px;
        height: 18px;
        border: 1px solid #bbb;
        margin-right: 8px;
        background-color: transparent;
        cursor: pointer;
        position: relative;
        flex-shrink: 0;
    }

    .search-checkbox input[type='checkbox']:checked::before {
        content: '✓';
        position: absolute;
        left: 50%;
        top: 50%;
        transform: translate(-50%, -50%);
        font-size: 14px;
        font-weight: bold;
    }

    .action-buttons-group {
        display: flex;
        align-items: center; /* Vertically align send button and file upload icon */
        gap: 8px; /* Space between send button and file upload icon */
        width: 100%;
    }

    .msger-send-btn {
        display: inline-flex;
        align-items: center;
        gap: 8px; /* Space between "Gửi" text and its SVG icon */
        cursor: pointer;
        padding: 6px 10px; /* Adjust padding to your preference */
        border: 1px solid #007bff; /* Example */
        background-color: #007bff; /* Example */
        color: white; /* Example */
        border-radius: 4px; /* Example */
        flex-shrink: 0; /* Prevent button from shrinking */
        /* Ensure text and icon are vertically centered if button height varies */
        line-height: 1;
    }

    .msger-send-btn:disabled {
        opacity: 0.6;
        cursor: not-allowed;
    }

    /* NEW: Styling for the label that acts as the file upload icon button */
    .file-upload-label {
        display: inline-flex; /* To allow padding and alignment */
        align-items: center;
        justify-content: center;
        padding: 6px; /* Adjust padding to make it look like an icon button */
        cursor: pointer;
        border-radius: 4px; /* Optional: match button's border-radius */
        /* Example styling for an icon button - adjust to your theme */
        border: 1px solid #ccc;
        background-color: #f0f0f0;
        color: #333; /* This will be the icon color if fill="currentColor" */
        transition:
            background-color 0.2s ease-in-out,
            border-color 0.2s ease-in-out;
    }

    .file-upload-label:hover {
        background-color: #e0e0e0;
        border-color: #bbb;
    }

    .file-upload-label .attach-icon {
        width: 20px; /* Adjust icon size as needed */
        height: 20px;
        fill: currentColor; /* Icon color inherits from label's text color */
    }

    /* NEW: Class to visually hide the actual file input */
    .visually-hidden-file-input {
        opacity: 0;
        position: absolute;
        width: 0.1px; /* Use small dimensions instead of 0 for some browser compatibilities */
        height: 0.1px;
        overflow: hidden;
        z-index: -1; /* Ensure it's behind everything */
        /* Alternative robust hiding technique:
  border: 0;
  clip: rect(0 0 0 0);
  height: 1px;
  margin: -1px;
  overflow: hidden;
  padding: 0;
  position: absolute;
  width: 1px;
  white-space: nowrap;
  */
    }

    /* === MOBILE RESPONSIVE STYLES === */
    @media (max-width: 768px) {
        .msger {
            margin: 0; /* Remove margin for full-width on mobile */
            height: 100vh; /* Full height for the chat container */
        }

        .msger-header {
            padding: 8px 10px; /* Adjust header padding */
            font-size: 1.1em; /* Slightly larger font for header */
        }

        .msger-chat {
            padding: 8px; /* Adjust chat padding */
        }

        .msger-inputarea {
            flex-direction: column; /* Stacks textarea and controls-group */
            padding: 10px; /* Overall padding for the input area on mobile */
            gap: 12px; /* Increased gap between textarea and controls-group */
        }

        .msger-input {
            /* Textarea */
            width: 100%;
            box-sizing: border-box;
            /* max-height: 70px; /* Base style - consider increasing for mobile */
            max-height: 100px; /* Allow more vertical space for typing on mobile */
            padding: 10px; /* Maintain comfortable padding */
            font-size: 1em; /* Or 0.95em if you prefer slightly smaller text */
        }

        .msger-input::placeholder {
            font-size: 1em; /* Adjust if base 1.5em is too large for mobile */
            /* opacity: 1; font-style: normal; /* Inherited or adjust as needed */
        }

        .controls-group {
            width: 100%;
            /* display: flex; flex-direction: column; align-items: flex-start; /* Inherited from base */
            gap: 12px; /* Increased gap between checkbox and action-buttons-group */
            /* padding-top: 10px; /* Removed, using gap on parent .msger-inputarea now */
        }

        .search-checkbox {
            /* display: flex; align-items: center; cursor: pointer; /* Inherited */
            font-size: 0.9em;
            padding: 4px 0; /* Add a little vertical space for easier tapping of the row */
        }

        .search-checkbox input[type='checkbox'] {
            width: 20px; /* Good size for touch */
            height: 20px;
            margin-right: 8px; /* Keep space from text */
        }
        .search-checkbox input[type='checkbox']:checked::before {
            font-size: 16px; /* Ensure checkmark is clear */
        }

        .msger-send-btn {
            flex-grow: 1; /* Allows button to take more space, useful for "Vui lòng chờ..." */
            justify-content: center; /* Centers text/icon if button grows */
            padding: 10px 12px; /* Generous padding for touch */
            font-size: 0.95em;
            /* line-height: 1; /* Base style, usually fine with flex align-items: center */
            /* Other visual styles (color, background, border) inherited or set here */
        }

        .file-upload-label {
            /* Attach icon button */
            padding: 10px; /* Match send button's vertical padding for consistent height */
            /* flex-shrink: 0; /* Default, good */
            /* Other visual styles inherited or set here */
        }
        .file-upload-label .attach-icon {
            width: 20px;
            height: 20px;
        }
    }

    @media (max-width: 480px) {
        .uploaded-file-preview-image {
            max-width: 150px;
            max-height: 120px;
        }

        .msger-inputarea {
            padding: 8px; /* Slightly reduce padding for very small screens */
            gap: 10px; /* Slightly reduce gap */
        }

        .msger-input {
            padding: 8px;
            font-size: 0.95em;
            max-height: 90px; /* Adjust max height if needed */
        }

        .msger-input::placeholder {
            font-size: 0.95em;
        }

        .controls-group {
            gap: 10px; /* Slightly reduce gap */
        }

        .search-checkbox {
            font-size: 0.85em;
        }
        .search-checkbox input[type='checkbox'] {
            width: 18px;
            height: 18px;
        }
        .search-checkbox input[type='checkbox']:checked::before {
            font-size: 14px;
        }

        .action-buttons-group {
            gap: 6px; /* Reduce gap between send and attach buttons */
        }

        .msger-send-btn {
            font-size: 0.9em;
            padding: 8px 10px;
        }

        .file-upload-label {
            padding: 8px;
        }
        .file-upload-label .attach-icon {
            width: 18px;
            height: 18px;
        }
    }

    /*
    ---------
     */

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    :root {
        --body-bg: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
        --msger-bg: #fff;
        --border: 2px solid #ddd;
        --left-msg-bg: #ececec;
        --left-msg-bg-dot: #999999;
        --right-msg-bg: #579ffb;
    }

    *,
    *:before,
    *:after {
        margin: 0;
        padding: 0;
        box-sizing: inherit;
    }

    .msger {
        display: flex;
        flex-flow: column wrap;
        justify-content: space-between;
        width: 100%;
        max-width: 967px;
        margin: 25px 10px;
        height: calc(100vh - 50px);
        border: var(--border);
        border-radius: 5px;
        background: var(--msger-bg);
        box-shadow: 0 15px 15px -5px rgba(0, 0, 0, 0.2);
    }

    .msger-header {
        display: flex;
        justify-content: space-between;
        padding: 10px;
        border-bottom: var(--border);
        background: #eee;
        color: #666;
    }

    .msger-chat {
        flex: 1;
        overflow-y: auto;
        padding: 10px;
    }
    .msger-chat::-webkit-scrollbar {
        width: 6px;
    }
    .msger-chat::-webkit-scrollbar-track {
        background: #ddd;
    }
    .msger-chat::-webkit-scrollbar-thumb {
        background: #bdbdbd;
    }
    .msg {
        display: flex;
        align-items: flex-end;
        margin-bottom: 10px;
        line-height: 1.5;
    }

    .msg:last-of-type {
        margin: 0;
    }

    .typing-indicator {
        display: flex;
        padding: 10px;
    }
    .dot {
        width: 8px;
        height: 8px;
        background: var(--left-msg-bg-dot);
        border-radius: 50%;
        margin: 0 2px;
        animation: bounce 1.5s infinite;
    }
    .dot:nth-child(2) {
        animation-delay: 0.2s;
    }

    .dot:nth-child(3) {
        animation-delay: 0.4s;
    }

    @keyframes bounce {
        0%,
        60%,
        100% {
            transform: translateY(0);
        }
        30% {
            transform: translateY(-5px);
        }
    }
    .msg-img {
        width: 50px;
        height: 50px;
        margin-right: 10px;
        /* background: #ddd; */
        background-repeat: no-repeat;
        background-position: center;
        background-size: cover;
        border-radius: 50%;
    }
    .msg-bubble {
        max-width: 850px;
        padding: 15px;
        border-radius: 15px;
        background: var(--left-msg-bg);
    }
    .msg-info {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 10px;
    }
    .msg-info-name {
        margin-right: 10px;
        font-weight: bold;
    }
    .msg-info-time {
        font-size: 0.85em;
    }

    .left-msg .msg-bubble {
        border-bottom-left-radius: 0;
    }

    .right-msg {
        flex-direction: row-reverse;
    }
    .right-msg .msg-bubble {
        background: var(--right-msg-bg);
        color: #fff;
        border-bottom-right-radius: 0;
    }
    .right-msg .msg-img {
        margin: 0 0 0 10px;
    }

    /* .msger-inputarea { */
    /*   display: flex; */
    /*   padding: 10px; */
    /*   border-top: var(--border); */
    /*   background: #eee; */
    /* } */

    :global(.msg-text pre) {
        max-width: 100%;
        overflow-x: auto;
        background: #222;
        color: #eee;
        padding: 12px;
        border-radius: 8px;
        font-size: 0.97em;
        margin: 8px 0;
        display: block;
    }

    :global(.right-msg .msg-text code) {
        background: rgba(255, 255, 255, 0.2);
        color: #ffeb3b;
        border: 1px solid rgba(255, 255, 255, 0.3);
    }

    :global(.msg-text code) {
        background: rgba(255, 255, 255, 0.1);
        color: #e91e63;
        padding: 2px 6px;
        border-radius: 4px;
        font-size: 0.9em;
        font-family: 'Courier New', Consolas, Monaco, monospace;
        border: 1px solid rgba(255, 255, 255, 0.2);
        font-weight: 500;
    }

    :global(.left-msg .msg-text code) {
        background: rgba(0, 0, 0, 0.1);
        color: #2196f3;
        border: 1px solid rgba(0, 0, 0, 0.2);
    }

    .msger-chat {
        background-color: #fcfcfe;
        background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='260' height='260' viewBox='0 0 260 260'%3E%3Cg fill-rule='evenodd'%3E%3Cg fill='%23dddddd' fill-opacity='0.4'%3E%3Cpath d='M24.37 16c.2.65.39 1.32.54 2H21.17l1.17 2.34.45.9-.24.11V28a5 5 0 0 1-2.23 8.94l-.02.06a8 8 0 0 1-7.75 6h-20a8 8 0 0 1-7.74-6l-.02-.06A5 5 0 0 1-17.45 28v-6.76l-.79-1.58-.44-.9.9-.44.63-.32H-20a23.01 23.01 0 0 1 44.37-2zm-36.82 2a1 1 0 0 0-.44.1l-3.1 1.56.89 1.79 1.31-.66a3 3 0 0 1 2.69 0l2.2 1.1a1 1 0 0 0 .9 0l2.21-1.1a3 3 0 0 1 2.69 0l2.2 1.1a1 1 0 0 0 .9 0l2.21-1.1a3 3 0 0 1 2.69 0l2.2 1.1a1 1 0 0 0 .86.02l2.88-1.27a3 3 0 0 1 2.43 0l2.88 1.27a1 1 0 0 0 .85-.02l3.1-1.55-.89-1.79-1.42.71a3 3 0 0 1-2.56.06l-2.77-1.23a1 1 0 0 0-.4-.09h-.01a1 1 0 0 0-.4.09l-2.78 1.23a3 3 0 0 1-2.56-.06l-2.3-1.15a1 1 0 0 0-.45-.11h-.01a1 1 0 0 0-.44.1L.9 19.22a3 3 0 0 1-2.69 0l-2.2-1.1a1 1 0 0 0-.45-.11h-.01a1 1 0 0 0-.44.1l-2.21 1.11a3 3 0 0 1-2.69 0l-2.2-1.1a1 1 0 0 0-.45-.11h-.01zm0-2h-4.9a21.01 21.01 0 0 1 39.61 0h-2.09l-.06-.13-.26.13h-32.31zm30.35 7.68l1.36-.68h1.3v2h-36v-1.15l.34-.17 1.36-.68h2.59l1.36.68a3 3 0 0 0 2.69 0l1.36-.68h2.59l1.36.68a3 3 0 0 0 2.69 0L2.26 23h2.59l1.36.68a3 3 0 0 0 2.56.06l1.67-.74h3.23l1.67.74a3 3 0 0 0 2.56-.06zM-13.82 27l16.37 4.91L18.93 27h-32.75zm-.63 2h.34l16.66 5 16.67-5h.33a3 3 0 1 1 0 6h-34a3 3 0 1 1 0-6zm1.35 8a6 6 0 0 0 5.65 4h20a6 6 0 0 0 5.66-4H-13.1z'/%3E%3Cpath id='path6_fill-copy' d='M284.37 16c.2.65.39 1.32.54 2H281.17l1.17 2.34.45.9-.24.11V28a5 5 0 0 1-2.23 8.94l-.02.06a8 8 0 0 1-7.75 6h-20a8 8 0 0 1-7.74-6l-.02-.06a5 5 0 0 1-2.24-8.94v-6.76l-.79-1.58-.44-.9.9-.44.63-.32H240a23.01 23.01 0 0 1 44.37-2zm-36.82 2a1 1 0 0 0-.44.1l-3.1 1.56.89 1.79 1.31-.66a3 3 0 0 1 2.69 0l2.2 1.1a1 1 0 0 0 .9 0l2.21-1.1a3 3 0 0 1 2.69 0l2.2 1.1a1 1 0 0 0 .9 0l2.21-1.1a3 3 0 0 1 2.69 0l2.2 1.1a1 1 0 0 0 .86.02l2.88-1.27a3 3 0 0 1 2.43 0l2.88 1.27a1 1 0 0 0 .85-.02l3.1-1.55-.89-1.79-1.42.71a3 3 0 0 1-2.56.06l-2.77-1.23a1 1 0 0 0-.4-.09h-.01a1 1 0 0 0-.4.09l-2.78 1.23a3 3 0 0 1-2.56-.06l-2.3-1.15a1 1 0 0 0-.45-.11h-.01a1 1 0 0 0-.44.1l-2.21 1.11a3 3 0 0 1-2.69 0l-2.2-1.1a1 1 0 0 0-.45-.11h-.01a1 1 0 0 0-.44.1l-2.21 1.11a3 3 0 0 1-2.69 0l-2.2-1.1a1 1 0 0 0-.45-.11h-.01zm0-2h-4.9a21.01 21.01 0 0 1 39.61 0h-2.09l-.06-.13-.26.13h-32.31zm30.35 7.68l1.36-.68h1.3v2h-36v-1.15l.34-.17 1.36-.68h2.59l1.36.68a3 3 0 0 0 2.69 0l1.36-.68h2.59l1.36.68a3 3 0 0 0 2.69 0l1.36-.68h2.59l1.36.68a3 3 0 0 0 2.56.06l1.67-.74h3.23l1.67.74a3 3 0 0 0 2.56-.06zM246.18 27l16.37 4.91L278.93 27h-32.75zm-.63 2h.34l16.66 5 16.67-5h.33a3 3 0 1 1 0 6h-34a3 3 0 1 1 0-6zm1.35 8a6 6 0 0 0 5.65 4h20a6 6 0 0 0 5.66-4H246.9z'/%3E%3Cpath d='M159.5 21.02A9 9 0 0 0 151 15h-42a9 9 0 0 0-8.5 6.02 6 6 0 0 0 .02 11.96A8.99 8.99 0 0 0 109 45h42a9 9 0 0 0 8.48-12.02 6 6 0 0 0 .02-11.96zM151 17h-42a7 7 0 0 0-6.33 4h54.66a7 7 0 0 0-6.33-4zm-9.34 26a8.98 8.98 0 0 0 3.34-7h-2a7 7 0 0 1-7 7h-4.34a8.98 8.98 0 0 0 3.34-7h-2a7 7 0 0 1-7 7h-4.34a8.98 8.98 0 0 0 3.34-7h-2a7 7 0 0 1-7 7h-7a7 7 0 1 1 0-14h42a7 7 0 1 1 0 14h-9.34zM109 27a9 9 0 0 0-7.48 4H101a4 4 0 1 1 0-8h58a4 4 0 0 1 0 8h-.52a9 9 0 0 0-7.48-4h-42z'/%3E%3Cpath d='M39 115a8 8 0 1 0 0-16 8 8 0 0 0 0 16zm6-8a6 6 0 1 1-12 0 6 6 0 0 1 12 0zm-3-29v-2h8v-6H40a4 4 0 0 0-4 4v10H22l-1.33 4-.67 2h2.19L26 130h26l3.81-40H58l-.67-2L56 84H42v-6zm-4-4v10h2V74h8v-2h-8a2 2 0 0 0-2 2zm2 12h14.56l.67 2H22.77l.67-2H40zm13.8 4H24.2l3.62 38h22.36l3.62-38z'/%3E%3Cpath d='M129 92h-6v4h-6v4h-6v14h-3l.24 2 3.76 32h36l3.76-32 .24-2h-3v-14h-6v-4h-6v-4h-8zm18 22v-12h-4v4h3v8h1zm-3 0v-6h-4v6h4zm-6 6v-16h-4v19.17c1.6-.7 2.97-1.8 4-3.17zm-6 3.8V100h-4v23.8a10.04 10.04 0 0 0 4 0zm-6-.63V104h-4v16a10.04 10.04 0 0 0 4 3.17zm-6-9.17v-6h-4v6h4zm-6 0v-8h3v-4h-4v12h1zm27-12v-4h-4v4h3v4h1v-4zm-6 0v-8h-4v4h3v4h1zm-6-4v-4h-4v8h1v-4h3zm-6 4v-4h-4v8h1v-4h3zm7 24a12 12 0 0 0 11.83-10h7.92l-3.53 30h-32.44l-3.53-30h7.92A12 12 0 0 0 130 126z'/%3E%3Cpath d='M212 86v2h-4v-2h4zm4 0h-2v2h2v-2zm-20 0v.1a5 5 0 0 0-.56 9.65l.06.25 1.12 4.48a2 2 0 0 0 1.94 1.52h.01l7.02 24.55a2 2 0 0 0 1.92 1.45h4.98a2 2 0 0 0 1.92-1.45l7.02-24.55a2 2 0 0 0 1.95-1.52L224.5 96l.06-.25a5 5 0 0 0-.56-9.65V86a14 14 0 0 0-28 0zm4 0h6v2h-9a3 3 0 1 0 0 6H223a3 3 0 1 0 0-6H220v-2h2a12 12 0 1 0-24 0h2zm-1.44 14l-1-4h24.88l-1 4h-22.88zm8.95 26l-6.86-24h18.7l-6.86 24h-4.98zM150 242a22 22 0 1 0 0-44 22 22 0 0 0 0 44zm24-22a24 24 0 1 1-48 0 24 24 0 0 1 48 0zm-28.38 17.73l2.04-.87a6 6 0 0 1 4.68 0l2.04.87a2 2 0 0 0 2.5-.82l1.14-1.9a6 6 0 0 1 3.79-2.75l2.15-.5a2 2 0 0 0 1.54-2.12l-.19-2.2a6 6 0 0 1 1.45-4.46l1.45-1.67a2 2 0 0 0 0-2.62l-1.45-1.67a6 6 0 0 1-1.45-4.46l.2-2.2a2 2 0 0 0-1.55-2.13l-2.15-.5a6 6 0 0 1-3.8-2.75l-1.13-1.9a2 2 0 0 0-2.5-.8l-2.04.86a6 6 0 0 1-4.68 0l-2.04-.87a2 2 0 0 0-2.5.82l-1.14 1.9a6 6 0 0 1-3.79 2.75l-2.15.5a2 2 0 0 0-1.54 2.12l.19 2.2a6 6 0 0 1-1.45 4.46l-1.45 1.67a2 2 0 0 0 0 2.62l1.45 1.67a6 6 0 0 1 1.45 4.46l-.2 2.2a2 2 0 0 0 1.55 2.13l2.15.5a6 6 0 0 1 3.8 2.75l1.13 1.9a2 2 0 0 0 2.5.8zm2.82.97a4 4 0 0 1 3.12 0l2.04.87a4 4 0 0 0 4.99-1.62l1.14-1.9a4 4 0 0 1 2.53-1.84l2.15-.5a4 4 0 0 0 3.09-4.24l-.2-2.2a4 4 0 0 1 .97-2.98l1.45-1.67a4 4 0 0 0 0-5.24l-1.45-1.67a4 4 0 0 1-.97-2.97l.2-2.2a4 4 0 0 0-3.09-4.25l-2.15-.5a4 4 0 0 1-2.53-1.84l-1.14-1.9a4 4 0 0 0-5-1.62l-2.03.87a4 4 0 0 1-3.12 0l-2.04-.87a4 4 0 0 0-4.99 1.62l-1.14 1.9a4 4 0 0 1-2.53 1.84l-2.15.5a4 4 0 0 0-3.09 4.24l.2 2.2a4 4 0 0 1-.97 2.98l-1.45 1.67a4 4 0 0 0 0 5.24l1.45 1.67a4 4 0 0 1 .97 2.97l-.2 2.2a4 4 0 0 0 3.09 4.25l2.15.5a4 4 0 0 1 2.53 1.84l1.14 1.9a4 4 0 0 0 5 1.62l2.03-.87zM152 207a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm6 2a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm-11 1a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm-6 0a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm3-5a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm-8 8a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm3 6a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm0 6a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm4 7a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm5-2a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm5 4a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm4-6a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm6-4a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm-4-3a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm4-3a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm-5-4a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm-24 6a1 1 0 1 1 2 0 1 1 0 0 1-2 0zm16 5a5 5 0 1 0 0-10 5 5 0 0 0 0 10zm7-5a7 7 0 1 1-14 0 7 7 0 0 1 14 0zm86-29a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm19 9a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-14 5a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-25 1a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm5 4a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm9 0a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm15 1a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm12-2a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-11-14a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-19 0a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm6 5a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-25 15c0-.47.01-.94.03-1.4a5 5 0 0 1-1.7-8 3.99 3.99 0 0 1 1.88-5.18 5 5 0 0 1 3.4-6.22 3 3 0 0 1 1.46-1.05 5 5 0 0 1 7.76-3.27A30.86 30.86 0 0 1 246 184c6.79 0 13.06 2.18 18.17 5.88a5 5 0 0 1 7.76 3.27 3 3 0 0 1 1.47 1.05 5 5 0 0 1 3.4 6.22 4 4 0 0 1 1.87 5.18 4.98 4.98 0 0 1-1.7 8c.02.46.03.93.03 1.4v1h-62v-1zm.83-7.17a30.9 30.9 0 0 0-.62 3.57 3 3 0 0 1-.61-4.2c.37.28.78.49 1.23.63zm1.49-4.61c-.36.87-.68 1.76-.96 2.68a2 2 0 0 1-.21-3.71c.33.4.73.75 1.17 1.03zm2.32-4.54c-.54.86-1.03 1.76-1.49 2.68a3 3 0 0 1-.07-4.67 3 3 0 0 0 1.56 1.99zm1.14-1.7c.35-.5.72-.98 1.1-1.46a1 1 0 1 0-1.1 1.45zm5.34-5.77c-1.03.86-2 1.79-2.9 2.77a3 3 0 0 0-1.11-.77 3 3 0 0 1 4-2zm42.66 2.77c-.9-.98-1.87-1.9-2.9-2.77a3 3 0 0 1 4.01 2 3 3 0 0 0-1.1.77zm1.34 1.54c.38.48.75.96 1.1 1.45a1 1 0 1 0-1.1-1.45zm3.73 5.84c-.46-.92-.95-1.82-1.5-2.68a3 3 0 0 0 1.57-1.99 3 3 0 0 1-.07 4.67zm1.8 4.53c-.29-.9-.6-1.8-.97-2.67.44-.28.84-.63 1.17-1.03a2 2 0 0 1-.2 3.7zm1.14 5.51c-.14-1.21-.35-2.4-.62-3.57.45-.14.86-.35 1.23-.63a2.99 2.99 0 0 1-.6 4.2zM275 214a29 29 0 0 0-57.97 0h57.96zM72.33 198.12c-.21-.32-.34-.7-.34-1.12v-12h-2v12a4.01 4.01 0 0 0 7.09 2.54c.57-.69.91-1.57.91-2.54v-12h-2v12a1.99 1.99 0 0 1-2 2 2 2 0 0 1-1.66-.88zM75 176c.38 0 .74-.04 1.1-.12a4 4 0 0 0 6.19 2.4A13.94 13.94 0 0 1 84 185v24a6 6 0 0 1-6 6h-3v9a5 5 0 1 1-10 0v-9h-3a6 6 0 0 1-6-6v-24a14 14 0 0 1 14-14 5 5 0 0 0 5 5zm-17 15v12a1.99 1.99 0 0 0 1.22 1.84 2 2 0 0 0 2.44-.72c.21-.32.34-.7.34-1.12v-12h2v12a3.98 3.98 0 0 1-5.35 3.77 3.98 3.98 0 0 1-.65-.3V209a4 4 0 0 0 4 4h16a4 4 0 0 0 4-4v-24c.01-1.53-.23-2.88-.72-4.17-.43.1-.87.16-1.28.17a6 6 0 0 1-5.2-3 7 7 0 0 1-6.47-4.88A12 12 0 0 0 58 185v6zm9 24v9a3 3 0 1 0 6 0v-9h-6z'/%3E%3Cpath d='M-17 191a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm19 9a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2H3a1 1 0 0 1-1-1zm-14 5a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-25 1a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm5 4a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm9 0a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm15 1a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm12-2a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-11-14a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-19 0a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm6 5a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-25 15c0-.47.01-.94.03-1.4a5 5 0 0 1-1.7-8 3.99 3.99 0 0 1 1.88-5.18 5 5 0 0 1 3.4-6.22 3 3 0 0 1 1.46-1.05 5 5 0 0 1 7.76-3.27A30.86 30.86 0 0 1-14 184c6.79 0 13.06 2.18 18.17 5.88a5 5 0 0 1 7.76 3.27 3 3 0 0 1 1.47 1.05 5 5 0 0 1 3.4 6.22 4 4 0 0 1 1.87 5.18 4.98 4.98 0 0 1-1.7 8c.02.46.03.93.03 1.4v1h-62v-1zm.83-7.17a30.9 30.9 0 0 0-.62 3.57 3 3 0 0 1-.61-4.2c.37.28.78.49 1.23.63zm1.49-4.61c-.36.87-.68 1.76-.96 2.68a2 2 0 0 1-.21-3.71c.33.4.73.75 1.17 1.03zm2.32-4.54c-.54.86-1.03 1.76-1.49 2.68a3 3 0 0 1-.07-4.67 3 3 0 0 0 1.56 1.99zm1.14-1.7c.35-.5.72-.98 1.1-1.46a1 1 0 1 0-1.1 1.45zm5.34-5.77c-1.03.86-2 1.79-2.9 2.77a3 3 0 0 0-1.11-.77 3 3 0 0 1 4-2zm42.66 2.77c-.9-.98-1.87-1.9-2.9-2.77a3 3 0 0 1 4.01 2 3 3 0 0 0-1.1.77zm1.34 1.54c.38.48.75.96 1.1 1.45a1 1 0 1 0-1.1-1.45zm3.73 5.84c-.46-.92-.95-1.82-1.5-2.68a3 3 0 0 0 1.57-1.99 3 3 0 0 1-.07 4.67zm1.8 4.53c-.29-.9-.6-1.8-.97-2.67.44-.28.84-.63 1.17-1.03a2 2 0 0 1-.2 3.7zm1.14 5.51c-.14-1.21-.35-2.4-.62-3.57.45-.14.86-.35 1.23-.63a2.99 2.99 0 0 1-.6 4.2zM15 214a29 29 0 0 0-57.97 0h57.96zM72.33 198.12c-.21-.32-.34-.7-.34-1.12v-12h-2v12a4.01 4.01 0 0 0 7.09 2.54c.57-.69.91-1.57.91-2.54v-12h-2v12a1.99 1.99 0 0 1-2 2 2 2 0 0 1-1.66-.88zM75 176c.38 0 .74-.04 1.1-.12a4 4 0 0 0 6.19 2.4A13.94 13.94 0 0 1 84 185v24a6 6 0 0 1-6 6h-3v9a5 5 0 1 1-10 0v-9h-3a6 6 0 0 1-6-6v-24a14 14 0 0 1 14-14 5 5 0 0 0 5 5zm-17 15v12a1.99 1.99 0 0 0 1.22 1.84 2 2 0 0 0 2.44-.72c.21-.32.34-.7.34-1.12v-12h2v12a3.98 3.98 0 0 1-5.35 3.77 3.98 3.98 0 0 1-.65-.3V209a4 4 0 0 0 4 4h16a4 4 0 0 0 4-4v-24c.01-1.53-.23-2.88-.72-4.17-.43.1-.87.16-1.28.17a6 6 0 0 1-5.2-3 7 7 0 0 1-6.47-4.88A12 12 0 0 0 58 185v6zm9 24v9a3 3 0 1 0 6 0v-9h-6z'/%3E%3Cpath d='M-17 191a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm19 9a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2H3a1 1 0 0 1-1-1zm-14 5a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-25 1a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm5 4a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm9 0a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm15 1a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm12-2a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-11-14a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-19 0a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm6 5a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-25 15c0-.47.01-.94.03-1.4a5 5 0 0 1-1.7-8 3.99 3.99 0 0 1 1.88-5.18 5 5 0 0 1 3.4-6.22 3 3 0 0 1 1.46-1.05 5 5 0 0 1 7.76-3.27A30.86 30.86 0 0 1-14 184c6.79 0 13.06 2.18 18.17 5.88a5 5 0 0 1 7.76 3.27 3 3 0 0 1 1.47 1.05 5 5 0 0 1 3.4 6.22 4 4 0 0 1 1.87 5.18 4.98 4.98 0 0 1-1.7 8c.02.46.03.93.03 1.4v1h-62v-1zm.83-7.17a30.9 30.9 0 0 0-.62 3.57 3 3 0 0 1-.61-4.2c.37.28.78.49 1.23.63zm1.49-4.61c-.36.87-.68 1.76-.96 2.68a2 2 0 0 1-.21-3.71c.33.4.73.75 1.17 1.03zm2.32-4.54c-.54.86-1.03 1.76-1.49 2.68a3 3 0 0 1-.07-4.67 3 3 0 0 0 1.56 1.99zm1.14-1.7c.35-.5.72-.98 1.1-1.46a1 1 0 1 0-1.1 1.45zm5.34-5.77c-1.03.86-2 1.79-2.9 2.77a3 3 0 0 0-1.11-.77 3 3 0 0 1 4-2zm42.66 2.77c-.9-.98-1.87-1.9-2.9-2.77a3 3 0 0 1 4.01 2 3 3 0 0 0-1.1.77zm1.34 1.54c.38.48.75.96 1.1 1.45a1 1 0 1 0-1.1-1.45zm3.73 5.84c-.46-.92-.95-1.82-1.5-2.68a3 3 0 0 0 1.57-1.99 3 3 0 0 1-.07 4.67zm1.8 4.53c-.29-.9-.6-1.8-.97-2.67.44-.28.84-.63 1.17-1.03a2 2 0 0 1-.2 3.7zm1.14 5.51c-.14-1.21-.35-2.4-.62-3.57.45-.14.86-.35 1.23-.63a2.99 2.99 0 0 1-.6 4.2zM15 214a29 29 0 0 0-57.97 0h57.96zM72.33 198.12c-.21-.32-.34-.7-.34-1.12v-12h-2v12a4.01 4.01 0 0 0 7.09 2.54c.57-.69.91-1.57.91-2.54v-12h-2v12a1.99 1.99 0 0 1-2 2 2 2 0 0 1-1.66-.88zM75 176c.38 0 .74-.04 1.1-.12a4 4 0 0 0 6.19 2.4A13.94 13.94 0 0 1 84 185v24a6 6 0 0 1-6 6h-3v9a5 5 0 1 1-10 0v-9h-3a6 6 0 0 1-6-6v-24a14 14 0 0 1 14-14 5 5 0 0 0 5 5zm-17 15v12a1.99 1.99 0 0 0 1.22 1.84 2 2 0 0 0 2.44-.72c.21-.32.34-.7.34-1.12v-12h2v12a3.98 3.98 0 0 1-5.35 3.77 3.98 3.98 0 0 1-.65-.3V209a4 4 0 0 0 4 4h16a4 4 0 0 0 4-4v-24c.01-1.53-.23-2.88-.72-4.17-.43.1-.87.16-1.28.17a6 6 0 0 1-5.2-3 7 7 0 0 1-6.47-4.88A12 12 0 0 0 58 185v6zm9 24v9a3 3 0 1 0 6 0v-9h-6z'/%3E%3Cpath d='M-17 191a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm19 9a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2H3a1 1 0 0 1-1-1zm-14 5a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-25 1a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm5 4a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm9 0a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm15 1a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm12-2a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm-11-14a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-19 0a1 1 0 0 0 0 2h2a1 1 0 0 0 0-2h-2zm6 5a1 1 0 0 1 1-1h2a1 1 0 0 1 0 2h-2a1 1 0 0 1-1-1zm-25 15c0-.47.01-.94.03-1.4a5 5 0 0 1-1.7-8 3.99 3.99 0 0 1 1.88-5.18 5 5 0 0 1 3.4-6.22 3 3 0 0 1 1.46-1.05 5 5 0 0 1 7.76-3.27A30.86 30.86 0 0 1-14 184c6.79 0 13.06 2.18 18.17 5.88a5 5 0 0 1 7.76 3.27 3 3 0 0 1 1.47 1.05 5 5 0 0 1 3.4 6.22 4 4 0 0 1 1.87 5.18 4.98 4.98 0 0 1-1.7 8c.02.46.03.93.03 1.4v1h-62v-1zm.83-7.17a30.9 30.9 0 0 0-.62 3.57 3 3 0 0 1-.61-4.2c.37.28.78.49 1.23.63zm1.49-4.61c-.36.87-.68 1.76-.96 2.68a2 2 0 0 1-.21-3.71c.33.4.73.75 1.17 1.03zm2.32-4.54c-.54.86-1.03 1.76-1.49 2.68a3 3 0 0 1-.07-4.67 3 3 0 0 0 1.56 1.99zm1.14-1.7c.35-.5.72-.98 1.1-1.46a1 1 0 1 0-1.1 1.45zm5.34-5.77c-1.03.86-2 1.79-2.9 2.77a3 3 0 0 0-1.11-.77 3 3 0 0 1 4-2zm42.66 2.77c-.9-.98-1.87-1.9-2.9-2.77a3 3 0 0 1 4.01 2 3 3 0 0 0-1.1.77zm1.34 1.54c.38.48.75.96 1.1 1.45a1 1 0 1 0-1.1-1.45zm3.73 5.84c-.46-.92-.95-1.82-1.5-2.68a3 3 0 0 0 1.57-1.99 3 3 0 0 1-.07 4.67zm1.8 4.53c-.29-.9-.6-1.8-.97-2.67.44-.28.84-.63 1.17-1.03a2 2 0 0 1-.2 3.7zm1.14 5.51c-.14-1.21-.35-2.4-.62-3.57.45-.14.86-.35 1.23-.63a2.99 2.99 0 0 1-.6 4.2zM15 214a29 29 0 0 0-57.97 0h57.96z'/%3E%3C/g%3E%3C/g%3E%3C/svg%3E");
    }
</style>
