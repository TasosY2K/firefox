# IMPORTANT_SCAN_OVERRIDE

This scan is a browser security review for Firefox and Chromium-family codebases.

Primary goal:
- Prioritize vulnerabilities that cross meaningful browser trust boundaries and could plausibly lead to severe impact in real deployments.
- Focus mainly on Critical findings, but allow reporting High-severity issues when they have a clear path to critical compromise or are strong candidates for realistic chaining.

Primary bug classes:
- Buffer overflow
- Heap overflow
- Use-after-free
- Null pointer dereference with plausible security impact
- Arbitrary JavaScript execution
- XSS
- UXSS
- CSRF when it crosses a meaningful browser or browser-adjacent trust boundary

In scope:
- Remote code execution from web content, network-originated data, or attacker-controlled browser inputs
- Sandbox, renderer, content-process, GPU-process, utility-process, broker, or privilege-boundary escape
- Memory corruption in parsing, rendering, media, graphics, networking, IPC, or developer tooling that can plausibly lead to code execution or sandbox escape
- Arbitrary JS execution in privileged or cross-origin contexts
- UXSS, origin confusion, site isolation bypass, and same-origin-policy failures with realistic attacker-controlled sources
- XSS in privileged browser surfaces such as internal pages, extension-adjacent pages, DevTools, browser UI, sync/admin surfaces, or debugging endpoints
- CSRF against browser-exposed management or debugging surfaces when it can trigger privileged state changes, local network access, code loading, or sensitive action execution
- URL, navigation, redirect, parser, serialization, storage, or IPC bugs that enable trusted code injection, origin confusion, privilege escalation, or secret exposure
- Information disclosure when it materially enables a higher-impact chain such as sandbox escape, credential theft, token extraction, or cross-origin compromise

Lower priority / usually out of scope:
- Pure renderer-only crashes with no plausible security consequence
- Null dereferences that are only stable denial of service with no realistic integrity or confidentiality impact
- Generic web-application XSS or CSRF that lives purely in a website rather than in browser code or browser-trusted surfaces
- Theoretical memory safety concerns without a credible attacker-controlled path
- Local-only issues that require privileges already equivalent to full user compromise

Repository-specific threat model for browsers:
- The browser is a multi-boundary platform that processes hostile content by default
- Key boundaries to evaluate:
  - web content to renderer/content process
  - renderer/content process to browser process
  - browser process to OS, filesystem, network, keychain, or device resources
  - web origin to other origins
  - untrusted page content to privileged browser UI or internal pages
  - untrusted input to developer, debugging, automation, or remote-control interfaces
  - network attacker to TLS, HTTP, WebSocket, QUIC, DNS, proxy, or update-related trust decisions
- Do not treat a simple renderer crash as high severity by itself unless there is a realistic route to control flow, sandbox escape, origin break, or privileged action
- Do consider realistic edge-case environments when they materially affect exploitability:
  - Windows path and font handling
  - Linux sandbox and namespace assumptions
  - macOS IPC, font, and graphics subsystem interactions
  - multi-process races
  - shared GPU or media processes
  - local debugging or remote debugging exposure
  - enterprise proxy environments
  - extension-enabled deployments

Priority review areas:

1. Memory corruption and crash-to-exploit candidates
- Inspect attacker-reachable parsing, layout, DOM, graphics, IPC, and media paths for:
  - buffer overflow
  - heap overflow
  - use-after-free
  - type confusion
  - uninitialized memory use
  - integer overflow that can realistically lead to memory corruption
- Prioritize cases where hostile web content, fonts, images, shaders, media, or network frames directly influence object lifetime, buffer sizing, or cross-process serialization

2. Arbitrary JavaScript execution, XSS, UXSS, and origin compromise
- Inspect parser, DOM, navigation, frame, worker, document, CSP, and internal-page logic for:
  - arbitrary JS execution in privileged contexts
  - cross-origin DOM access
  - origin confusion
  - scheme confusion
  - XSS in browser-trusted pages
  - UXSS from parser or process-isolation failures
- Prioritize cases where attacker-controlled content can execute script in:
  - `chrome://`, `about:`, `devtools://`, extension, settings, sync, account, or browser UI surfaces
  - a more-privileged frame or process
  - another site/origin without user-intended navigation

3. IPC, sandbox, and process-boundary escape
- Inspect Firefox IPC/IPDL/PBackground and Chromium Mojo/IPC/broker patterns for:
  - missing validation
  - confused deputy behavior
  - dangerous deserialization
  - lifetime bugs across process boundaries
  - privileged operations reachable from compromised low-trust processes
- Prioritize cases where attacker-controlled renderer content can reach filesystem, network, device, debugging, or OS-level capabilities not intended for web content

4. Networking, transport, and protocol trust
- Inspect TCP/IP, HTTP, HTTP/2, HTTP/3, QUIC, WebSocket, proxy, TLS, DNS, and certificate handling
- Prioritize:
  - origin confusion
  - cross-protocol confusion
  - credential leakage
  - request smuggling or parser differentials across browser components
  - websocket or transport bugs that enable privileged or cross-origin actions
  - certificate or hostname validation issues that enable code/resource injection or session compromise

5. Privileged tooling and debugging surfaces
- Inspect CDP, DevTools, remote debugging, browser automation, and browser-internal debugging flows
- Prioritize:
  - CSRF against exposed debugging endpoints
  - origin bypass in DevTools or debugging pages
  - unauthenticated or weakly authenticated local network exposure
  - message confusion between web content and debugging transports
  - privileged code execution paths triggered from inspected content

6. Files, archives, fonts, and local resource loading
- Inspect font parsing, file URLs, downloaded artifacts, local resource loaders, temporary files, and cache interactions
- Prioritize:
  - hostile font parsing leading to memory corruption
  - local file access or cross-origin file disclosure
  - cache poisoning or resource confusion that upgrades web content into a more trusted context
  - overwrite or local persistence paths through browser-managed files

Evidence expectations for each finding:
- State the trust boundary crossed:
  - remote unauthenticated web content
  - network attacker / MITM
  - compromised renderer or content process
  - lower-privilege local user
  - untrusted extension-adjacent or debugging-adjacent input
  - cross-origin or cross-site boundary
- Show the source -> sink path with file/function references
- List concrete preconditions, such as:
  - attacker-controlled HTML/CSS/JS/SVG/XML/XSLT/font/media/shader input
  - WebRTC reachable from attacker-controlled page
  - DevTools or CDP exposed locally or remotely
  - WebSocket or QUIC reachable with attacker-controlled frames
  - browser internal page or privileged scheme reachable
  - compromised renderer with IPC access
- Explain the end impact clearly:
  - code execution
  - sandbox escape
  - trusted UI/script injection
  - UXSS or cross-origin data theft
  - privileged state change via CSRF
  - credential, cookie, token, or key material exposure enabling broader takeover
- If reporting a chain, describe each step and why it is realistic in deployed browsers

Chaining guidance:
- Actively look for realistic chains, not only standalone bugs
- In-scope examples:
  - web content parser bug -> heap corruption -> renderer code execution
  - renderer compromise -> IPC validation flaw -> sandbox escape
  - origin confusion -> privileged page JS execution -> browser account/token theft
  - DevTools or CDP exposure -> CSRF or auth bypass -> arbitrary browser control
  - QUIC, WebSocket, or HTTP parser bug -> memory corruption or trust confusion -> privileged action
  - font or graphics parsing bug -> GPU or renderer memory corruption -> process escape
  - XML/XSLT bug -> privileged script execution or cross-origin read -> account/session compromise
- If a finding is not Critical on its own, it may still be reportable if the chain to higher impact is concrete and realistic

Practical audit bias:
- Start with the most exposed and historically relevant browser surfaces:
  - CSS
  - fonts
  - WebRTC
  - audio/media
  - WebGL and graphics
  - IPC and process messaging
  - WebSockets
  - TCP/IP stack
  - CDP and DevTools
  - QUIC / HTTP/3
  - XML / XSLT
- Then review realistic environment edge cases:
  - Windows font and path behavior
  - GPU process and driver-facing assumptions
  - local DevTools exposure
  - cross-process races and lifetime mismatches
  - enterprise TLS/proxy environments
  - site isolation and cross-origin transitions
- Finally, assess whether identified issues can lead to:
  - code execution
  - sandbox or broker escape
  - UXSS or privileged XSS
  - trusted browser action via CSRF
  - persistent compromise through browser-managed state

## Firefox-Specific Override

Primary Firefox component focus:
- CSS: Stylo, layout style resolution, animation, selector matching, computed style and DOM/style interaction
- Fonts: font loading, OpenType/WOFF/WOFF2 parsing, glyph shaping, text layout, HarfBuzz/graphics-adjacent boundaries
- WebRTC: signaling-adjacent parsing, SDP, RTP/RTCP, ICE, SCTP/DataChannel, media pipeline interactions, content-to-network transitions
- Audio/Media: demuxers, codecs, MSE, WebAudio, AudioIPC, stream graph lifetime, decoder process or utility process boundaries
- WebGL and Graphics: canvas, WebGL, WebGPU if present, shaders, Skia/WebRender/ANGLE-adjacent paths, GPU process or compositor interaction
- IPC: IPDL, PBackground, content-parent/content-child, JS actor surfaces, serialization and lifetime validation
- WebSockets: frame parsing, compression, extension negotiation, origin handling, auth/cookie propagation
- TCP/IP stack: `netwerk`, DNS, HTTP, cache, speculative connections, proxy, TLS integration
- DevTools and remote debugging: DevTools server, browser toolbox, remote debugging toggles, privileged pages and message channels
- QUIC/HTTP3: `necko` HTTP/3 and QUIC parsing, stream state machines, alt-svc, coalescing, origin/auth handling
- XML/XSLT: DOMParser, XML parser paths, XSLT transforms, document privilege transitions, script execution or external resource handling

Firefox-specific bug patterns to prioritize:
- UAF or type confusion from DOM/layout/style/tree mutation races
- Privileged script execution through `about:` or browser UI document confusion
- Content process to parent process escalation through IPDL validation bugs
- Font or graphics parsing memory corruption reachable from web content
- WebRTC message or state-machine bugs that cross from web content into network or privileged media handling
- `netwerk` origin, redirect, cache, or auth confusion enabling UXSS, credential leakage, or privileged request execution
- DevTools server or remote debugging features reachable from local network, localhost rebinding, or hostile pages

Firefox evidence expectations:
- Name the specific actor or IPC route when the bug crosses a process boundary
- State whether the impacted context is content, parent, GPU/compositor, media, socket, or privileged browser UI
- Show whether the bug lands in `dom/`, `layout/`, `gfx/`, `netwerk/`, `toolkit/`, `browser/`, `security/`, or an IPC definition path

## Chromium-Specific Override

Primary Chromium component focus:
- CSS: Blink style engine, selector processing, layout tree, animation, style invalidation, DOM/CSS interaction
- Fonts: font parsing/loading, FreeType/DirectWrite/CoreText-adjacent paths, shaping, font fallback, PDF or renderer font handling
- WebRTC: SDP, ICE, SCTP, RTP/RTCP, peer-connection lifetime, Mojo handoff between renderer/browser/network/media paths
- Audio/Media: media parsers, demuxers, codecs, audio services, WebAudio, renderer-to-utility or renderer-to-GPU transitions
- WebGL and Graphics: Blink graphics, ANGLE, Dawn/WebGPU if present, Skia, Viz, GPU command buffers, shader translation and IPC
- IPC and Mojo: renderer-browser-GPU-network utility service boundaries, interface brokers, deserialization, validation, capability scoping
- WebSockets: frame parsing, permessage-deflate, auth propagation, origin enforcement, browser/network service transitions
- TCP/IP stack: `net`, DNS, proxy, TLS, cache, socket pools, speculative fetches, HTTP parser and connection reuse behavior
- CDP and DevTools: remote debugging port, DevTools frontend/backend messaging, inspected-page to tooling trust boundaries
- QUIC/HTTP3: `net/quic`, HTTP/3 parsing, stream and control frame state, alt-svc, coalescing, certificate/origin behavior
- XML/XSLT: Blink XML parser, XSLT processing, document creation, privileged scheme transitions, extension or internal-page interaction

Chromium-specific bug patterns to prioritize:
- UAF, heap overflow, or type confusion in Blink, media, graphics, or Mojo-exposed paths reachable from web content
- Renderer-to-browser or renderer-to-privileged-service escalation through Mojo validation gaps
- UXSS from site isolation mistakes, navigation confusion, or scheme/origin mismatches
- DevTools or CDP CSRF, origin bypass, or auth weakness that grants arbitrary browser control
- Network service confusion enabling cross-origin response mixups, credential leakage, or trusted resource loading
- GPU command or shader-related memory corruption reachable from WebGL/WebGPU/canvas content

Chromium evidence expectations:
- Name the exact process boundary crossed: renderer, browser, GPU, utility, network, or storage/service process
- Identify the specific Mojo interface, browser service, or Blink entry point when applicable
- Show whether the vulnerable path lives in `content/`, `components/`, `chrome/`, `net/`, `services/`, `gpu/`, `media/`, `ui/`, `skia/`, or Blink/WebKit-adjacent paths

## Component-Specific Review Prompts

### CSS
- Look for style recalculation races, lifetime bugs, malformed selector parsing, and cross-document or cross-origin style application issues
- Prioritize crashers with attacker-controlled DOM mutation, animation timing, or style invalidation

### Fonts
- Look for malformed font parsing, table length/integer issues, glyph shaping corruption, and browser/graphics process trust crossings
- Prioritize web font inputs that reach native parsers or privileged processes

### WebRTC
- Look for parser confusion in SDP/ICE/STUN/TURN/RTP, state-machine bugs, message reordering, and renderer-to-network/media privilege crossings
- Prioritize peer-controlled packets or signaling fields that influence memory ownership or privileged operations

### Audio / Media
- Look for demuxer, decoder, resampler, and stream-lifetime bugs
- Prioritize remotely supplied containers or streams that cross into utility, media, or GPU processes

### WebGL / Graphics
- Look for shader translation bugs, command-buffer validation failures, unsafe memory sharing, and GPU process trust failures
- Prioritize paths where untrusted shaders, textures, or command sequences reach native code or privileged GPU services

### IPC / Mojo / IPDL
- Look for missing argument validation, type confusion in deserialization, stale handles, race conditions, and confused deputy patterns
- Prioritize routes from compromised renderers/content processes into browser-privileged operations

### WebSockets
- Look for frame length bugs, compression issues, auth/cookie leakage, cross-origin or mixed-scheme confusion, and handshake parser differentials
- Prioritize cases that transform attacker-controlled frames into privileged requests or privileged browser actions

### TCP/IP Stack
- Look for parser inconsistencies, connection reuse confusion, DNS/proxy auth edge cases, and trust failures around redirects, coalescing, and credential reuse
- Prioritize issues that enable cross-origin data mixups, credential theft, or trusted resource injection

### CDP / DevTools
- Look for exposed debugging ports, CSRF, weak origin checks, localhost trust mistakes, and messaging paths from web content into privileged tooling
- Prioritize findings that grant page inspection, cookie/token access, filesystem access, or arbitrary browser automation

### QUIC / HTTP3
- Look for malformed frame handling, state-machine confusion, cross-origin connection coalescing mistakes, alt-svc trust issues, and memory corruption in parser paths
- Prioritize issues that can be triggered pre-auth from hostile servers or network attackers

### XML / XSLT
- Look for parser differential issues, external resource loading, transform-time script execution, origin inheritance bugs, and privileged document confusion
- Prioritize pathways from attacker-controlled XML/XSLT into privileged script execution, UXSS, or sensitive data exposure
