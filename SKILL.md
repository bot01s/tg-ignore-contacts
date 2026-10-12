---
name: tg-ignore-contacts
description: Add or re-apply the local "ignore contacts" feature to Telegram Desktop. A profile button adds a user to ~/.config/tdesktop/tg_cont_ignore.conf, and that user's messages are removed from the main group chat feed unless they reply to me or @mention me. A chat-menu checkbox reveals them again. Also clones or actualizes the tdesktop checkout together with its git submodules and the tdesktop:centos_env dependency image when they are missing or out of date. Use when the user asks to add, prepare for, port, rebase, re-apply, debug or extend the tg-ignore-contacts feature.
---

# Ignore Contacts

Local-only (never sent to the server) feature. A user toggled to "ignored" has
their messages hidden in group chats of the main chat view.

There is no patch file. The feature is re-created from the declarative
instructions below, which name the files to locate, the anchors to find in them
and what must be true after the change. Locate every anchor by searching for
the quoted symbol, never by line number: upstream moves code constantly.

## Prepare the checkout

Do this first, once per session. It makes sure there is a complete, current
checkout of `https://github.com/telegramdesktop/tdesktop` with the expected
branch, all pinned submodules and the Docker dependency image. It is safe to
re-run.

Set `REPO` to the checkout root: the **current directory** (`.`) of the
workspace where tdesktop is checked out or should be checked out. Never use an
absolute path. In a workspace that also holds other files (for example this
one), clone into a `tdesktop/` subfolder and use `REPO=tdesktop`. Set `BRANCH`
to the branch the feature targets (default `dev`, or the branch the user names).

### 1. Obtain or actualize the repository

- If `$REPO` is not a git checkout (for example an empty workspace), clone it
  **without submodules**, then initialize them (step 2). Use `.` when the
  current directory is empty, else a `tdesktop/` subfolder:

  ```bash
  git clone --branch "$BRANCH" \
    https://github.com/telegramdesktop/tdesktop.git "$REPO"
  ```

- If `$REPO` is already a checkout, actualize it **without discarding local
  work**. Confirm it is clean first:

  ```bash
  cd "$REPO"
  git status --porcelain
  ```

  - If the output is **empty**, fast-forward onto the upstream branch:

    ```bash
    git fetch origin "$BRANCH"
    git checkout "$BRANCH"
    git merge --ff-only "origin/$BRANCH"
    ```

  - If the output is **not empty**, local changes exist: do **not** reset,
    clean or stash them, and do not switch branches. Leave the tree as it is
    unless the user explicitly asks for a clean/actualized checkout, and then
    confirm which local changes may be discarded before anything destructive.

  Never force-reset or `git clean` this checkout as part of this workflow: the
  feature is applied on top of the current tree.

### 2. Initialize and update submodules

Pinned submodules must match the merged commit, even in a checkout that was
cloned without `--recursive`:

```bash
cd "$REPO"
git submodule sync --recursive
git submodule update --init --recursive
```

Never use `git submodule update --remote`: the dependencies must stay pinned
to the commit the superproject records.

### 3. Ensure the dependency image

The C++ dependencies (Qt, FFmpeg, OpenSSL, and the rest) are not sibling
folders. They live inside the Docker image `tdesktop:centos_env`, built from
`Telegram/build/docker/centos_env/Dockerfile`; the build mounts this checkout
at `/usr/src/tdesktop` and links those libraries statically.

- If `docker image inspect tdesktop:centos_env` succeeds, reuse the image. A
  freshly merged `dev` can bump a dependency, so if the build later fails to
  find a library (or `Telegram/build/docker/centos_env/Dockerfile` changed in
  the merged commits), rebuild the image.
- To (re)build it on a Linux host, run:

  ```bash
  cd "$REPO"
  Telegram/build/prepare/linux.sh
  ```

  This needs `poetry` and a working Docker daemon and downloads/builds a large
  toolchain (tens of GB, many minutes). Do not run it unless the image is
  missing or the superproject Dockerfile changed; ask the user first when
  unsure. Confirm the daemon with `docker info` before building or compiling.

- **The checked-in `Dockerfile` is a Jinja template, not a buildable file.**
  Building it directly fails with `gcc: unrecognized debug output level 'z{%'`
  because the `{% if ... %}` markers leak into `CFLAGS`. When `poetry` is not
  installed, render it yourself and build the rendered file instead:

  ```bash
  cd "$REPO/Telegram/build/docker/centos_env"
  python3 gen_dockerfile.py > /tmp/tdesktop-centos_env.Dockerfile
  DOCKER_BUILDKIT=1 docker build -t tdesktop:centos_env \
    -f /tmp/tdesktop-centos_env.Dockerfile .
  ```

  `gen_dockerfile.py` needs `jinja2` (`pip3 install --user jinja2`). The
  rendered file has no `COPY`/`ADD` from the build context, so the context
  directory does not matter.

### 4. Report what was done

Say concisely whether the checkout was cloned, fast-forwarded or left
untouched; whether submodules were initialized/updated; and whether the
dependency image was reused or rebuilt.

### 5. Confirm the build credentials

The build must use the real `api_id`/`api_hash`. **Using
`TDESKTOP_API_TEST=ON` and the test-only credentials `17349` /
`344583e45741c457fe1862106095a5eb` is forbidden**, even when the current
`out/` tree was configured with them - reconfigure it instead (see **API
credentials** below). Never silence a missing-credentials error by enabling the
test flag or by passing the test values.

Before building, check the credentials file exists (`~/.tdesktop-api.txt`, see
**API credentials**). **If it is missing, stop and ask the user for the real
`api_id`/`api_hash`** and tell them to put them into `~/.tdesktop-api.txt`
themselves; do not proceed and do not fall back to the test credentials.

## Repository skills

The checkout carries its own agent skills under `$REPO/.agents/skills/`. They
are part of the repository, not of this skill, and they encode the project's
own workflows. **Read and follow them whenever the task falls into their
domain** - do not re-invent their steps from this file.

- `$REPO/.agents/skills/rebase/SKILL.md` - intent-aware rebase of the current
  checkout. Use it whenever the user asks to rebase, port or re-apply this
  feature onto a new base, or to continue/abort an in-progress rebase, instead
  of resolving conflicts by hand.
- `$REPO/.agents/skills/perform-task/SKILL.md` - implement, review, test and
  publish exactly one existing `ai-tdesktop` task. Use it when the work is
  owned by a queue task (see `AGENTS.md`), not for an ordinary conversational
  request.
- `$REPO/.agents/skills/continue/SKILL.md` - the checkout-level scheduler that
  drains the AI queue. Use it when the user asks to continue autonomous work.
- `$REPO/.agents/skills/process-inbox/SKILL.md` - triage the ignored
  `ai-tdesktop/inbox/inbox.md` into task records. Planning only, never
  implementation.
- `$REPO/.agents/skills/dependency-watch/SKILL.md` - audit dependencies of
  `origin/dev`. Advisory only; it does not build or commit.

Shared references live in `$REPO/.agents/shared/` (for example
`build-lock-recovery.md`, `phase-effort.md`, `test-loop.md`); the skills above
link to them and expect them to be read when they say so. `AGENTS.md` remains
the authority on style, commits and the AI-task queue; these skills are the
procedures that operate on top of it.

## Behavior spec

- Profile of any user (not self/support/verify-codes): toggle button
  "Ignore" / "Stop ignoring" (next to "Block user").
- Storage: `~/.config/tdesktop/tg_cont_ignore.conf`
  (`QStandardPaths::GenericConfigLocation` + `/tdesktop/`). One user per line:
  `<user_id> <name> @username`; `#` lines are comments. Read once at startup,
  rewritten on every toggle. The list is global for all accounts.
- Hidden only in groups: `peer->isChat() || peer->isMegagroup()`. Channels and
  private chats are untouched. Notifications are untouched.
- Still visible from an ignored user: messages that `@mention` me
  (`item->mentionsMe()`), and replies to my own messages (resolved reply
  target `from()->isSelf()`). Own (`out()`) and service messages are never hidden.
- Chat three-dots menu in a group: checkbox "Show ignored messages", shown only
  when the ignore list is non-empty. State is per chat and in-memory only.
- Main menu (hamburger) entry "Ignore list", between "Calls" and "Saved
  Messages" (hidden in support mode). It opens a box listing every ignored
  user with name, userpic and `@username` (or id); clicking a row opens the
  profile, the "Stop ignoring" link on the right removes the user. The box
  refreshes live when the list changes elsewhere.
- Toggling any control re-lays-out the already loaded feed immediately.

## Implementation prompt

First run **Prepare the checkout** above. Then implement the feature below in
this checkout. Work through the numbered steps in order, search for each quoted
anchor, make exactly the described change, and do not touch anything else.
Follow `AGENTS.md` (tabs, no new comments beyond one short line, `auto`, `_q`
literals, no hardcoded sizes, no `Q_OS_LINUX`). If the work is owned by an
`ai-tdesktop` queue task, drive it through `$REPO/.agents/skills/perform-task`
(see **Repository skills**); if the user asks to port or re-apply the feature
onto a new base, use `$REPO/.agents/skills/rebase`.

### 1. Create `Telegram/SourceFiles/core/core_ignore.h` and `core_ignore.cpp`

Add `class Ignore final` in `namespace Core` with these members and nothing
that talks to the network:

- State: `base::flat_set<UserId> _ids`; `base::flat_set<PeerId> _revealed`
  (per-chat "show ignored", memory only); `base::flat_map<UserId, QString>
  _names` and `_usernames` (text remembered from the conf file);
  `rpl::event_stream<> _changes`.
- `load()`: clear all three id/name maps, open the conf for reading as text, skip
  blank and `#` lines, take the first whitespace-delimited token as the numeric
  id (skip the line if it is `0`), then treat a trailing token that starts with
  `@` and follows whitespace as the username and the remaining text as the name.
  A missing file is not an error.
- `save()`: `mkpath` the directory, rewrite the file with a two-line `#` header
  (`# Telegram Desktop ignored contacts` and `# <user_id> <name> @username`)
  and one `id name @username` line per id. For each id, prefer the loaded peer
  (`App().maybePrimarySession()` then `session->data().peerLoaded(
  peerFromUser(id))`) and update `_names`/`_usernames` from it when non-empty,
  otherwise keep the remembered values. An id must never lose its remembered
  name. Write the id with `id.bare` (`UserId` has no `.value`).
- `toggle(UserId)`, `setIgnored(UserId, bool)` (no-op when unchanged, else
  `save()` and fire `_changes`), `setRevealed(PeerId, bool)` (no-op when
  unchanged, else fire `_changes`, no save).
- Const accessors: `contains(UserId)`, `empty()`, `ids()`, `revealed(PeerId)`,
  `changes()`, `savedName(UserId)`, `savedUsername(UserId)`; static
  `FilePath()`.
- Path: `QStandardPaths::writableLocation(GenericConfigLocation) + "/tdesktop/"
  + "tg_cont_ignore.conf"`. Do not use `AppConfigLocation` (that is
  `TelegramDesktop`, not `tdesktop`).

Register both files in `Telegram/CMakeLists.txt` in the `core/` block, in
alphabetical order: insert `core/core_ignore.cpp` / `core/core_ignore.h`
directly between `core/core_cloud_password.h` and
`core/core_screenshot_protection.cpp` (do not anchor on
`core/core_settings.cpp`, which would place them after
`core_screenshot_protection.*` and break the alphabetical order).

### 2. Own it in `Core::Application`

- `core/application.h`: forward-declare `class Ignore;` beside
  `class Settings;` in `namespace Core`, and declare
  `[[nodiscard]] Ignore &ignore();` right after the `settings()` overloads.
- `core/application.cpp`: include `core/core_ignore.h`; add a member
  `Ignore ignore;` to `struct Application::Private` (after `Settings settings;`);
  define `Application::ignore()` returning `_private->ignore`; in
  `Application::run()` call `_private->ignore.load();` immediately after
  `startLocalStorage();`.

### 3. Profile button: `info/profile/info_profile_actions.cpp`

- Include `core/core_ignore.h`.
- In `class ActionsFiller` declare `void addIgnoreAction(not_null<UserData*>
  user);` after `addBlockAction`.
- Implement it right after `ActionsFiller::addBlockAction`, modelled on that
  function: compute `const auto id = peerToUser(user->id);` (never use
  `user->id` directly, it is a `PeerId`), build a label producer from
  `rpl::single(rpl::empty) | rpl::then(Core::App().ignore().changes())`
  mapped to `tr::lng_profile_unignore_user()` when `contains(id)` else
  `tr::lng_profile_ignore_user()`, flattened with `rpl::flatten_latest()` and
  `rpl::start_spawning(_wrap->lifetime())`; derive `toggleOn` the same way
  `addBlockAction` does; call `AddActionButton(_wrap, text, toggleOn, [=] {
  Core::App().ignore().toggle(id); }, &st::infoIconBlock,
  st::infoBlockButton)`.
- In `ActionsFiller::fillUserActions`, call `addIgnoreAction(user);` directly
  after `addBlockAction(user);` inside the same
  `!isSelf() && !isSupport() && !isVerifyCodes()` condition.

### 4. Hide messages: `history/view/history_view_element.{h,cpp}`

This is the hook that matters. The ordinary group chat is drawn by the legacy
`HistoryWidget`/`HistoryInner` using `History::blocks`; do not filter slices in
`ChatWidget::historySource()` (that path serves only topics, replies and
sublists and silently does nothing here).

- Header: declare `[[nodiscard]] bool isHiddenByIgnore() const;` next to
  `isHiddenByGroup()`.
- Source: include `core/core_ignore.h`. Change `Element::isHidden()` to
  `return isHiddenByGroup() || isHiddenByIgnore();`.
- Define `Element::isHiddenByIgnore()`: return `false` unless
  `_context == Context::History` and the ignore list is non-empty; `false` for
  `item->out()` and `item->isService()`; `false` unless
  `item->history()->peer` is `isChat()` or `isMegagroup()` and is not
  `revealed(peer->id)`; `false` unless `item->from()->asUser()` exists and
  `contains(peerToUser(user->id))`; `false` if `item->mentionsMe()`; if the item
  has `HistoryMessageReply` with a non-null `resolvedMessage`, return
  `!resolvedMessage->from()->isSelf()`; otherwise `true`.
  Do not add another flag: `isHidden()` already collapses the view to zero
  height (`Message::marginTop/marginBottom/resizeContentGetHeight`) and is
  skipped by `previousDisplayedInBlocks()`/`nextDisplayedInBlocks()`.

### 5. Re-layout already loaded views: `history/history_widget.cpp`

- Include `core/core_ignore.h`.
- In the `HistoryWidget` constructor, immediately before the
  `session().data().viewHeightAdjusted()` subscription (search for the existing
  `viewResizeRequest` subscription just above it), subscribe to
  `Core::App().ignore().changes()` with `rpl::on_next(..., lifetime())`. For
  each of `_history` and `_migrated` (skip null), walk every
  `history->blocks[i]->messages[j]` view and call
  `previousInBlocksChanged()` then `setPendingResize()`; afterwards, if
  `_history && _list`, call `updateHistoryGeometry()`. `setPendingResize()`
  alone leaves stale date separators.

### 6. "Show ignored messages" checkbox: `window/window_peer_menu.cpp`

- Include `core/core_ignore.h`.
- In `class Filler` declare `void addToggleShowIgnored();` after
  `addToggleTopicClosed`.
- Implement it after `Filler::addToggleNoForwards`: return early unless `_peer`
  is `isChat()` or `isMegagroup()` and `!Core::App().ignore().empty()`;
  capture `revealed = ignore().revealed(peer->id)`; add the action with
  `_addAction({ .make = ... })` where `make` builds a
  `Menu::ItemWithCheck(menuParent->menu(), menuParent->menu()->st(),
  Ui::Menu::CreateAction(menuParent->menu().get(),
  tr::lng_context_show_ignored(tr::now), callback), nullptr, nullptr)`, calls
  `item->init(revealed)` and returns it. The callback flips
  `setRevealed(peer->id, !revealed(peer->id))` and then
  `weak.get()->hideMenu()` for a `base::make_weak(menuParent)`.
- In `Filler::fillHistoryActions` call `addToggleShowIgnored();` right after
  `addInfo();` (and before `addViewAsTopics();`).

### 7. Ignore list box: create `boxes/ignore_list_box.h` and `.cpp`

- Header: `[[nodiscard]] object_ptr<Ui::BoxContent> PrepareIgnoreListBox(
  not_null<Window::SessionController*> window);`
- Source, in an anonymous namespace:
  - `IgnoreRow : PeerListRowWithLink` (from `boxes/peer_list_controllers.h`)
    holding a fallback name; override `generateName()` to return the base name,
    or the fallback when the base name is empty.
  - `IgnoreListController : PeerListController` holding the
    `SessionController`, an `MTP::Sender _api(&window->session().mtp())` and a
    `base::flat_set<UserId> _requested`. `session()` returns the window's
    session. `prepare()` sets the title `tr::lng_menu_ignore_list()`, calls
    `refresh()` and `requestMissing()`, and re-runs both on every
    `Core::App().ignore().changes()` event (subscription on `lifetime()`).
    `rowClicked` does `crl::on_main(window, [=] { window->showPeerInfo(peer); })`.
    `rowRightActionClicked` calls
    `Core::App().ignore().setIgnored(peerToUser(row->peer()->id), false)`.
  - `createRow(UserData*)`: build an `IgnoreRow` with
    `ignore().savedName(id)` as fallback, `setActionLink(
    tr::lng_profile_unignore_user(tr::now))`, and `setCustomStatus` to
    `@username` (live `user->username()`, else `savedUsername(id)`) or the
    decimal id when no username is known (use `id.bare`, not `id.value` -
    here `id` is a `UserId`).
  - `refresh()`: remove rows whose user is no longer in `ignore().ids()`
    (iterate backwards with `peerListRowAt`), append a row for each id that has
    none (`session().data().user(id)`, find by `user->id.value`), set
    `setDescriptionText` to empty when there are rows else
    `tr::lng_ignore_list_empty(tr::now)`, then `peerListRefreshRows()`.
  - `requestMissing()`: for each id whose `session().data().user(id)->name()`
    is empty and which is not in `_requested`, remember it and collect
    `user->inputUser()` into a `QVector<MTPInputUser>` (not
    `std::vector`); if any, send `MTPusers_GetUsers(
    MTP_vector<MTPInputUser>(std::move(ids)))`, and in `.done` call
    `session().data().processUsers(result)`, remove every row and `refresh()`
    again so names and userpics appear.
  - `PrepareIgnoreListBox` returns `Box<PeerListBox>(std::move(controller),
    init)` where `init` adds a `tr::lng_close()` button that closes the box.
- Register `boxes/ignore_list_box.cpp/.h` in `Telegram/CMakeLists.txt` after
  `boxes/gift_premium_box.h`.

### 8. Main menu entry: `window/window_main_menu.cpp`

- Include `boxes/ignore_list_box.h`.
- In `MainMenu::setupMenu()`, inside the non-support branch, add right after the
  `lng_menu_calls` action and before the `lng_saved_messages` action:
  `addAction(tr::lng_menu_ignore_list(), { &st::menuIconBlock })->
  setClickedCallback([=] { controller->show(PrepareIgnoreListBox(controller)); });`

### 9. Strings: `Resources/langs/lang.strings`

Add after `"lng_profile_unblock_user"`:

```
"lng_profile_ignore_user" = "Ignore";
"lng_profile_unignore_user" = "Stop ignoring";
"lng_context_show_ignored" = "Show ignored messages";
"lng_menu_ignore_list" = "Ignore list";
"lng_ignore_list_empty" = "No ignored users yet. Use the Ignore button in a user's profile.";
```

## Critical gotchas (learned the hard way)

1. **Hook `Element::isHidden()`, not the message slice** (see step 4). A filter
   in `ChatWidget` compiles, links, and does nothing in a regular group.
2. Changing the hidden state of already built views needs
   `previousInBlocksChanged()` plus `setPendingResize()`, then
   `updateHistoryGeometry()` (step 5).
3. `UserData::id` is a `PeerId`; `Core::Ignore` keys on `UserId`. Always
   convert with `peerToUser(user->id)`.
4. `rpl::before_next` is a pass-through filter and cannot change a value; use
   `rpl::map` if a producer's value must change.
5. The conf directory is `~/.config/tdesktop/` literally; build it from
   `GenericConfigLocation`. `GenericConfigLocation` is cross-platform, so no
   `Q_OS_*` guard is needed. Never use `Q_OS_LINUX` (see `AGENTS.md`).
6. Reply-to-me detection relies on `HistoryMessageReply::resolvedMessage`; if
   the replied-to message is not loaded, the ignored user's reply stays hidden.
7. `session().data().user(id)` creates an empty record for unknown ids. That is
   why the list box needs `users.getUsers` and the conf-saved name fallback.
8. Do not add `#ifdef _DEBUG` code or debug state to production files.
9. `PeerId` exposes the raw number as `.value`, but `UserId` (a
   `ChatIdType<0>`) exposes it as `.bare`. Use the right one: `user->id.value`
   (`PeerId`) is fine, `id.value` on a `UserId` will not compile.
10. `base::flat_map` has no `.value()` and no `.at()`; read it with `find()`
    and compare the iterator to `end()`.
11. `MTP_vector<T>` is constructed from a `QVector<T>`, not a `std::vector<T>`;
    accumulate into a `QVector` before building the MTP vector.

## Known limitations

- Unread counters and the chat-list preview still count hidden messages.
- Not applied to channels, comment threads, topics, pinned view, search,
  chat previews or the media viewer.
- The "show ignored" state resets on restart.
- The list file is not watched; edit it only while the app is closed.

## Build and verify

Follow `AGENTS.md`: Debug only, never Release. The dependency image
`tdesktop:centos_env` must be present (see **Prepare the checkout** step 3;
build it with `Telegram/build/prepare/linux.sh` if it is missing). Linux/WSL
builds then go through the Docker environment, for example:

```bash
docker run --rm -i -u $(id -u) -v "$PWD:/usr/src/tdesktop" -e CONFIG=Debug \
  tdesktop:centos_env /usr/src/tdesktop/Telegram/build/docker/centos_env/build.sh \
  -D TDESKTOP_API_ID=<id> -D TDESKTOP_API_HASH=<hash>
```

### API credentials (`api_id` / `api_hash`)

They are baked in at configure time through `-D TDESKTOP_API_ID` and
`-D TDESKTOP_API_HASH`, never read at runtime. On this machine they are kept
outside the repository:

- Credentials file: `~/.tdesktop-api.txt` (mode `600`), two lines:

  ```
  api_id: <number>
  api_hash: <hex string>
  ```

  Override the location with the `TDESKTOP_CREDS` environment variable.
- Build wrapper: `~/.tdesktop-build.sh`. It parses that file with `sed`, then
  runs the `docker run ... build.sh -D TDESKTOP_API_ID=... -D TDESKTOP_API_HASH=...`
  command above for the checkout root (the current directory) with
  `CONFIG=Debug`. It exits with an error if the file is missing or unparsable.
- Rebuild after code changes: just run `~/.tdesktop-build.sh`.

Rules:

- **Never enable `TDESKTOP_API_TEST` and never use the shipped test-only
  credentials** `api_id: 17349` / `api_hash:
  344583e45741c457fe1862106095a5eb` (or `docs/api_credentials.md`). The option is
  defined in `Telegram/cmake/telegram_options.cmake`; when `ON` it force-sets
  those test values, overriding any `-D TDESKTOP_API_ID` / `-D TDESKTOP_API_HASH`
  you pass, so a tree with `TDESKTOP_API_TEST:BOOL=ON` in `out/CMakeCache.txt`
  silently ignores the real credentials. A build for this feature builds and
  runs the real client, so the test credentials are off-limits here.
- **If `~/.tdesktop-api.txt` (or the wrapper) is missing, you are required to
  ask the user for the credentials** and tell them to put the values into
  `~/.tdesktop-api.txt` themselves. Do not build, do not guess, and do not fall
  back to the test credentials. Never read the file's contents into the
  conversation, and do not echo, log or paste the values into any command,
  commit or note.
- Before building, make sure the configured tree does not carry the test
  credentials. `grep -E 'TDESKTOP_API_(ID|HASH)|TDESKTOP_API_TEST' \
  out/CMakeCache.txt` must not show `TDESKTOP_API_TEST:BOOL=ON` nor
  `TDESKTOP_API_ID:STRING=0`. If it does, reconfigure the tree with
  `-DTDESKTOP_API_TEST=OFF -DTDESKTOP_API_ID=<id> -DTDESKTOP_API_HASH=<hash>`
  (or wipe and reconfigure `out/`) before building.
- Never create the file inside the repository (an untracked
  `api-id-api-hash.txt` in the repo root is one `git add .` away from a leak).
- Never put real API credentials in the repository, a commit, or this skill.

Manual check in a group chat:

1. Open a user's profile, press "Ignore"; the label becomes "Stop ignoring"
   and the id appears in `tg_cont_ignore.conf`.
2. That user's already loaded messages vanish from the group feed with no gap
   and no orphan date header.
3. Their reply to one of my messages and a message that `@mentions` me remain.
4. Chat menu -> "Show ignored messages" brings everything back; unchecking
   hides it again.
5. "Stop ignoring" restores the user's messages.
6. Hamburger menu -> "Ignore list" shows each ignored user with name, userpic
   and `@username`; "Stop ignoring" there removes the row and restores that
   user's messages.
7. The same user's private chat and channel posts are unaffected.

## Commit

One commit, subject like `Add local ignore list for group chat messages`, with
the two-line `Task:` trailer only if an ai-tdesktop task owns it. Not an `[ai]`
commit: it changes product code. Do not add comments beyond what `AGENTS.md`
allows.
