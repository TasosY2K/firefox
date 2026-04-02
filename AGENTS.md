# Firefox Audit Branch

This branch is a pruned audit view of Firefox for autonomous code review and vulnerability discovery.

Rules:
- Treat missing roots as intentionally pruned noise, not as dead upstream code.
- Escalate to main only when a kept file directly depends on code that is absent here.
- Prioritize privilege boundaries, parsers, IPC, networking, storage, sandboxing, and browser-to-content transitions.

High-value roots:
- rowser/
- dom/
- ipc/
- js/
- layout/
- media/
- 
etwerk/
- security/
- 	oolkit/
- widget/
- xpcom/

Pruned at the root on this branch:
- 	hird_party/
- 	esting/
- 	ools/
- devtools/
- 	askcluster/
- docs/
- uild/
- mobile/
- servo/
- 
sprpub/
"@

 = @"
# Audit Scope

Purpose: reduce audit noise while preserving the primary first-party Firefox attack surface.

Kept areas:
- browser UI and feature code
- DOM, JS, layout, IPC, networking, storage, media, and security code
- core platform glue needed for source navigation

Removed at the root:
- vendored dependencies and licensing bundles
- large test harnesses and CI/task orchestration
- developer tooling, docs, Android/mobile packaging, and Servo-side trees

Important limitation:
- nested vendor or test directories inside kept roots may still exist; use the ignore files in this branch to deprioritize them.