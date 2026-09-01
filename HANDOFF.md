# Dopamine 3.0.9 + Roothide — Handoff

Cập nhật: 2026-09-01. Session cũ hết / user chuyển chat mới. File này để tiếp tục.

## 1. Mục tiêu

Port roothide (Dopamine2-roothide) lên opa334/Dopamine 3.0.9 cho chip/iOS mới.

Mục tiêu cuối: **1 bản jailbreak roothide hoàn chỉnh**:

- Vẫn ẩn root như roothide 2.x: jbroot random `/var/containers/Bundle/Application/.jbroot-XXXXXXXX`
- `/var/jb` không phải root thật, chỉ là symlink theo mô hình roothide
- dpkg trong jbroot dùng Architecture `iphoneos-arm64e`
- Có RootHide Manager / roothideapp, package managers, basebin-link, libkrw, libroot
- Sideload bằng developer cert được, không bắt buộc TrollStore
- Clean uninstall
- Chỉ build bằng GitHub Actions CI, không có macOS local
- Phải kiểm tra không thiếu phần quan trọng của Dopamine 3 rootless gốc và Dopamine 2.x roothide gốc

## 2. Repo / branch / remote

- Working dir chính: `/root/claude_workspace/dopamine-roothide-merged`
- Remote `roothide`: `https://github.com/huyff0247-source/dopamine-3-roothide`
- Remote `upstream`: `https://github.com/opa334/Dopamine.git`
- Branch local: `merge-upstream`
- Push CI: `git push roothide merge-upstream:main`
- Repo so sánh 2.x gốc: `/root/claude_workspace/dopamine-roothide-base`
- Repo upstream 3.x nếu cần: `/root/claude_workspace/dopamine-upstream`

Luôn dùng `--repo huyff0247-source/dopamine-3-roothide` với `gh`.

Working tree hiện tại:

- `HANDOFF.md` untracked. Không commit trừ khi user bảo.
- `BaseBin/opainject` dirty submodule. Không đụng, không commit.
- HEAD hiện tại: `f5c97417`

## 3. CI

Workflow quan trọng:

- `.github/workflows/roothide.yml` = `*** build tipa file ***`
- Artifact: `roothide-Dopamine-<VERSION>.tipa`
- Runner: macos-15, Xcode latest-stable (thực tế Xcode 16), THEOS roothide

`.github/workflows/main.yml` vẫn fail vì Xcode 16 clash `xpc_private.h`:

```text
../.include/xpc_private.h:18:21: error: conflicting types for 'xpc_dictionary_copy_mach_send'
```

Ignore main.yml. Chỉ cần tipa workflow success.

Build-fix loop:

```bash
cd /root/claude_workspace/dopamine-roothide-merged
git push roothide merge-upstream:main
gh run list --repo huyff0247-source/dopamine-3-roothide --limit 8 --json databaseId,status,conclusion,headSha,workflowName,displayTitle,createdAt
gh run view <ID> --repo huyff0247-source/dopamine-3-roothide --log-failed | tail -100
```

Không sleep timeout dài. Poll `gh run list` mỗi ~30s. Khi tipa workflow `completed success` mới được báo build xong.

CI gần nhất:

| Commit | Workflow | Run ID | Kết quả |
|---|---:|---:|---|
| `f5c97417` | tipa | `33485931673` | success |
| `f5c97417` | main.yml | `33485931670` | failure, xpc clash, ignore |
| `d8ed4658` | tipa | `33481090520` | success |
| `c288e10a` | tipa | `33476230180` | success |
| `0fe87c72` | tipa | `33472146007` | success |

## 4. Thiết bị test

- iPhone12,5
- iOS 18.6.2, build 22G100
- CF major = 19, bootstrap dùng `bootstrap_1900.tar.zst`
- Sideload bằng developer cert

## 5. Bootstrap roothide đang bundle

Đã commit 2 bootstrap 2.x vào repo:

```text
Application/Dopamine/Resources/bootstrap_1800.tar.zst
Application/Dopamine/Resources/bootstrap_1900.tar.zst
```

Chúng là bootstrap roothide 2.x thật, không phải vanilla Procursus. Có:

- `./prep_bootstrap.sh`
- `./usr/bin/dpkg` arch roothide
- dpkg Architecture `iphoneos-arm64e`
- Các symlink:
  - `./private/var -> ../var`
  - `./var/tmp -> ../tmp`

Không được để workflow download vanilla Procursus nữa. `download_bootstraps.sh` vẫn còn trong cây nhưng workflow không gọi. Có thể xóa sau nếu muốn dọn, nhưng không cần cho fix hiện tại.

## 6. Lịch sử commit quan trọng

| Commit | Nội dung |
|---|---|
| `46c88bd5` | mkdir `sources.list.d` trước khi write `default.sources` |
| `7f598bfb` | giữ `basebin.tc` trong jbroot, không xóa |
| `3657ee7e` | guard `fixBootstrapSymlink` khi symlink thiếu hoặc không phải jbroot |
| `b5c0884b` | skip `updatelinks.sh` trong nhánh roothide finalizeBootstrap |
| `8b8d3e27` | build Packages debs trước xcodebuild để bundle vào tipa |
| `0fe87c72` | build BaseBin trước Packages để có `libjailbreak.h` |
| `c288e10a` | sửa libkrw-provider/basebin-link cho bootstrap roothide: arch `iphoneos-arm64e`, path trong `.package/usr/...` |
| `d8ed4658` | bundle bootstrap roothide 2.x ~19MB, bỏ bước download vanilla Procursus |
| `f5c97417` | luôn replace symlink `private/var` và `tmp` khi extract bootstrap |

## 7. Trạng thái jailbreak hiện tại

Bản user test mới nhất: commit `f5c97417`, CI tipa run `33485931673`.

Kết quả trên device:

```text
Exploit OK
PPL OK
Phys R/W OK
Elevate OK
remove unknown/unfinished jbroot OK
device is not strapped...
bootstrap @ /var/containers/Bundle/Application/.jbroot-F01DEB6FC4898FAB
Extracting Bootstrap
Status: Bootstrap Installed
Status: Bootstrap Successful
Updating BaseBin
Loading BaseBin TrustCache
Initializing Environment / inject launchd OK
Bind Mount Done
roothider_main.c:66: failed ASSERTion `roothidehooks != NULL'
Finalizing Bootstrap
roothider_main.c:66: failed ASSERTion `roothidehooks != NULL'
Jailbreak failed with error: BootstrapErrorDomain Code=-6 "prep_bootstrap.sh returned 6"
```

Tiến bộ so với bản trước:

- Không còn lỗi `Failed to install the libkrw plugin: 2`
- Không còn lỗi `errno 17 File exists` ở `private/var`
- Bootstrap extract xong
- BaseBin trustcache load được
- launchd inject được
- bind mount xong

Lỗi chặn hiện tại nằm sau bind mount / finalize bootstrap.

## 8. Root cause hiện tại — gần chắc

### 8.1 `roothidehooks.dylib` thiếu trong basebin

`BaseBin/systemhook/src/roothider_main.c` line 61-70:

```c
void loadPathHook()
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        void* roothidehooks = dlopen(JBROOT_PATH("/basebin/roothidehooks.dylib"), RTLD_NOW);
        ASSERT(roothidehooks != NULL);
        void (*pathhook)() = dlsym(roothidehooks, "pathhook");
        ASSERT(pathhook != NULL);
        pathhook();
    });
}
```

Log device báo:

```text
roothider_main.c:66: failed ASSERTion `roothidehooks != NULL'
```

Nghĩa là `dlopen(JBROOT_PATH("/basebin/roothidehooks.dylib"))` fail. Khả năng cao file `roothidehooks.dylib` không được đóng gói vào `basebin.tar`.

Bằng chứng trong merged repo:

- Có source: `BaseBin/roothidehooks/`
- Có Makefile Theos: `BaseBin/roothidehooks/Makefile`
- `BaseBin/Makefile` line 12:

```make
subprojects: ChOma XPF MachOMerger opainject libjailbreak systemhook forkfix launchdhook hookd boomerang jbctl idownloadd watchdoghook rootlesshooks dopamine
```

Không có `roothidehooks`.

- `BaseBin/Makefile` line 82-84:

```make
rootlesshooks: .build .include libjailbreak
	$(MAKE) -C rootlesshooks
	@cp rootlesshooks/.theos/obj/rootlesshooks.dylib .build
```

Không có target tương tự cho `roothidehooks`.

Do đó `basebin.tar` nhiều khả năng chỉ có `rootlesshooks.dylib`, thiếu `roothidehooks.dylib`. Systemhook roothide cần `roothidehooks.dylib` để gọi `pathhook()`.

### 8.2 `prep_bootstrap.sh returned 6`

`prep_bootstrap.sh` trong bootstrap:

```sh
#!/bin/sh

/usr/libexec/firmware
/usr/sbin/pwd_mkdb -p /etc/master.passwd >/dev/null 2>&1
/Library/dpkg/info/debianutils.postinst configure 99999
/Library/dpkg/info/apt.postinst configure 999999
/Library/dpkg/info/dash.postinst configure 999999
/Library/dpkg/info/zsh.postinst configure 999999
/Library/dpkg/info/bash.postinst configure 999999
/Library/dpkg/info/vi.postinst configure 999999

/usr/sbin/pwd_mkdb -p /etc/master.passwd

/usr/bin/chsh -s /usr/bin/zsh mobile
/usr/bin/chsh -s /usr/bin/zsh root

if [ -z "$NO_PASSWORD_PROMPT" ]; then
    PASSWORDS=""
    PASSWORD1=""
    PASSWORD2=""
    while [ -z "$PASSWORD1" ] || [ ! "$PASSWORD1" = "$PASSWORD2" ]; do
            PASSWORDS="$(/usr/bin/uialert -b ...)"
            PASSWORD1="$(printf "%s\n" "$PASSWORDS" | /usr/bin/sed -n '1 p')"
            PASSWORD2="$(printf "%s\n" "$PASSWORDS" | /usr/bin/sed -n '2 p')"
    done
    printf "%s\n" "$PASSWORD1" | /usr/sbin/pw usermod 501 -h 0
fi

rm -f /prep_bootstrap.sh
```

Exit code 6 chưa giải thích chắc chắn. Giả thuyết:

1. `roothidehooks` thiếu làm path redirection không đúng khi spawn script/con của script. Các path `/usr/sbin/pwd_mkdb`, `/Library/dpkg/info/*.postinst` có thể không resolve vào jbroot.
2. Script trả exit code từ một lệnh nào đó; cần capture stdout/stderr thật của `prep_bootstrap.sh` để biết chính xác.
3. Có thể liên quan `NO_PASSWORD_PROMPT`: nếu không set, script chờ `uialert`; nhưng log fail code 6 không giống chờ UI.

Ưu tiên fix 8.1 trước vì nó là lỗi rõ ràng và giống root cause của path hook.

## 9. Fix cần làm tiếp

### Fix chính: build và bundle `roothidehooks.dylib`

Sửa `BaseBin/Makefile`:

1. Thêm `roothidehooks` vào `subprojects`.
2. Thêm target:

```make
roothidehooks: .build .include libjailbreak
	$(MAKE) -C roothidehooks
	@cp roothidehooks/.theos/obj/roothidehooks.dylib .build
```

3. Thêm vào `.PHONY` nếu cần.

Lưu ý: `BaseBin/roothidehooks/Makefile` dùng:

```make
roothidehooks_CFLAGS = -Werror -fobjc-arc -I../.include -Wno-nonportable-include-path
roothidehooks_INSTALL_PATH = /basebin
```

`-Wno-nonportable-include-path` đã có sẵn trong merged, khác 2.x một chút, không sao.

Sau khi build, verify trong artifact:

```bash
gh run download <TIPA_RUN_ID> --repo huyff0247-source/dopamine-3-roothide -D /tmp/tipa_check
cd /tmp/tipa_check
unzip -o *.tipa -d tipa_unpacked
# tipa là IPA-like; Payload/Dopamine.app/basebin.tar hoặc đường tương tự
tar -tvf Payload/Dopamine.app/basebin.tar | grep roothidehooks
```

Cần thấy:

```text
basebin/roothidehooks.dylib
```

Nếu không thấy, fix chưa đúng.

### Fix phụ nếu vẫn còn exit 6

Sau khi có `roothidehooks.dylib`, nếu `prep_bootstrap.sh` vẫn trả 6:

1. Thêm log stdout/stderr cho `exec_cmd_trusted` hoặc capture output riêng.
2. Kiểm tra merged có set `NO_PASSWORD_PROMPT` khi finalize không. So sánh 2.x:

```bash
grep -R "NO_PASSWORD_PROMPT" /root/claude_workspace/dopamine-roothide-base
grep -R "NO_PASSWORD_PROMPT" /root/claude_workspace/dopamine-roothide-merged
```

3. Kiểm tra `exec_cmd_trusted(JBROOT_PATH("/bin/sh"), "/prep_bootstrap.sh", NULL)` trong merged vs 2.x. Merged line ~979, 2.x line ~1280.

## 10. Những lỗi device đã vượt qua

Không được đoán lại các lỗi này:

1. EPERM mkdir `.jbroot` khi sideload. Fix: `proc_csflags_set(proc, CS_INSTALLER)` trong `elevatePrivileges`. Commit cũ trước `f5c97417`.
2. ENOENT `private/var`, `tmp` khi extract vanilla bootstrap. Fix bằng tolerant mkdir/move và sau đó bundle bootstrap roothide.
3. ENOENT write `default.sources`. Fix: mkdir `sources.list.d`.
4. `fixBootstrapSymlink` errno 2. Fix: return 0 khi ENOENT.
5. `updatelinks.sh returned 2`. Fix: skip trong roothide branch.
6. `Failed to load BaseBin trustcache`. Fix: không xóa `basebin.tc`.
7. `Failed to install the libkrw plugin: 2`. Fix: arch/path debs + bundle bootstrap roothide.
8. `errno 17 File exists` khi tạo symlink `private/var`. Fix: unconditional remove rồi tạo lại, commit `f5c97417`.

## 11. Điều không được làm

- Không commit `HANDOFF.md` trừ khi user yêu cầu.
- Không đụng `BaseBin/opainject` dirty submodule.
- Không đổi `Packages/libroot` sang arch `iphoneos-arm64e` hoặc path `/usr/lib`. libroot phải giữ:
  - control Architecture `iphoneos-arm64`
  - package path `.package/var/jb/usr/lib/libroot.dylib`
  vì `finalizeBootstrap` unpack bằng `dpkg-deb -R` rồi copy từ `/var/jb/usr/lib/libroot.dylib` vào `jbrootPrefix(@"/usr/lib/libroot.dylib")`.
- Không nói log fail là của build cũ nếu user xác nhận đó là build mới.
- Không dùng vanilla Procursus bootstrap.
- Không thêm bước download bootstrap vào workflow.

## 12. Chi tiết package layout roothide

### libkrw-provider

- control Architecture: `iphoneos-arm64e`
- package files dưới `.package/usr/lib/libkrw/...`
- rpath nên dùng `@loader_path/.jbroot/usr/lib` hoặc layout tương thích roothide

### basebin-link

- control Architecture: `iphoneos-arm64e`
- package:
  - `.package/usr/bin/jbctl -> ../../basebin/jbctl`
  - `.package/usr/bin/opainject -> ../../basebin/opainject`
  - `.package/usr/lib/libjailbreak.dylib -> ../../basebin/libjailbreak.dylib`

### libroot

- control Architecture: `iphoneos-arm64`
- package: `.package/var/jb/usr/lib/libroot.dylib`
- Không đổi.

### roothideapp

- Chỉ cài khi `prep_bootstrap.sh` tồn tại trong jbroot.
- Path bundle: `roothideapp.deb` trong app bundle.
- Đây là RootHide Manager.

## 13. finalizeBootstrap hiện tại trong merged

File: `Application/Dopamine/Jailbreak/DOBootstrapper.m`

Khoảng line 975:

```objc
if ([[NSFileManager defaultManager] fileExistsAtPath:jbrootPrefix(@"/prep_bootstrap.sh")]) {
    [[DOUIManager sharedInstance] sendLog:@"Finalizing Bootstrap" debug:NO];
    int r = exec_cmd_trusted(JBROOT_PATH("/bin/sh"), "/prep_bootstrap.sh", NULL);
    if (r != 0) {
        return error prep_bootstrap.sh returned r;
    }

    installPackageManagers;
    install roothideapp.deb;
    remove SplashBoard snapshots;
}
else {
    Updating Symlinks;
    fixBootstrapSymlink /bin/sh /usr/bin/sh;
    skip updatelinks.sh;
}

install libkrw-dopamine.deb if needed;
install basebin-link.deb if needed;
unpack libroot via dpkg-deb -R and copy to jbroot /usr/lib/libroot.dylib;
```

Bản mới nhất đã đi đúng nhánh có `prep_bootstrap.sh`, tức bootstrap roothide đã được nhận diện.

## 14. Quy trình làm việc user mong đợi

- Trả lời tiếng Việt nếu user viết tiếng Việt.
- Caveman full mode đang bật: ngắn, bỏ từ thừa, giữ thuật ngữ kỹ thuật nguyên.
- Commit message tiếng Anh.
- Không hỏi lặp lại nếu đã có đủ context.
- User có thể treo máy: tự làm, tự build, tự chờ CI success, rồi mới dừng nếu không còn gì làm.
- Mỗi khi đổi trạng thái quan trọng, cập nhật `HANDOFF.md`.
- Trước khi dừng hoặc chuyển session, `HANDOFF.md` phải phản ánh đúng lỗi mới nhất.

## 15. Việc cần làm ngay khi session mới bắt đầu

1. Đọc file này.
2. Sửa `BaseBin/Makefile` để build `roothidehooks.dylib` và copy vào `.build`.
3. Commit bằng tiếng Anh, ví dụ:

```text
fix: build roothidehooks.dylib into basebin for roothide pathhook
```

4. Push:

```bash
git push roothide merge-upstream:main
```

5. Chờ tipa workflow success.
6. Download artifact, verify `basebin.tar` có `roothidehooks.dylib`.
7. Báo user test device.
8. Nếu vẫn `prep_bootstrap.sh returned 6`, capture output của script và kiểm tra `NO_PASSWORD_PROMPT`, path redirection, postinst paths.

## 16. Quick commands

```bash
cd /root/claude_workspace/dopamine-roothide-merged

git status
git log --oneline -15
git push roothide merge-upstream:main

gh run list --repo huyff0247-source/dopamine-3-roothide --limit 8 --json databaseId,status,conclusion,headSha,workflowName,displayTitle,createdAt
gh run view <RUN_ID> --repo huyff0247-source/dopamine-3-roothide --log-failed | tail -120

gh run download <RUN_ID> --repo huyff0247-source/dopamine-3-roothide -D /tmp/tipa_check
```

Kiểm tra bootstrap tar:

```bash
zstd -dc Application/Dopamine/Resources/bootstrap_1900.tar.zst | tar -tv | grep -E ' \./(private|tmp|var)(/|$| )'
zstd -dc Application/Dopamine/Resources/bootstrap_1900.tar.zst | tar -xO ./prep_bootstrap.sh
```

Kiểm tra basebin.tar trong tipa:

```bash
cd /tmp/tipa_check
unzip -o *.tipa -d tipa_unpacked
find tipa_unpacked -name basebin.tar -print
tar -tvf <path-to-basebin.tar> | grep -E 'roothidehooks|rootlesshooks|basebin.tc|systemhook'
```
