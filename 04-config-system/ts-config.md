
  
  init.el (Doom 模块)
  
  (doom! :tools
         lsp                       ; ★ 默认 lsp-mode (不要 +eglot)
         debugger                  ; 可选
  
         :checkers
         syntax                    ; flycheck
  
         :editor
         (format +onsave)          ; ★ Doom 内置 format 模块,自动 format
  
         :lang
         (javascript +typescript +tree-sitter +lsp)   ; ★ 正确位置
         (web +html +css)          ; HTML/CSS (TSX 不需要,但配套常用)
         json                      ; tsconfig.json
         markdown
         sh)
  
  关键差异:
  - (javascript +typescript +tree-sitter +lsp) — 不是 (web ...)
  - (format +onsave) — 不是装 format-all
  - lsp 不带 +eglot — 用 lsp-mode
  
  改完 doom sync。
  
  装 LSP server (2026 推荐)
  
  # 方案 1: typescript-language-server (兼容性最好,文档最多)
  bun add -d typescript typescript-language-server
  
  # 方案 2: vtsls (性能更好, Zed / LazyVim 默认)
  bun add -d vtsls typescript
  
  2026 选择:
  - 新项目、追求速度 → vtsls (Zed 和 LazyVim 都默认它)
  - 老项目、要稳 → typescript-language-server (lsp-mode 文档最多)
  
  别单独用 Biome 当 LSP——它是 lint+format,配 LSP 用:
  bun add -d @biomejs/biome   # lint + format (替代 ESLint + Prettier)
  
  config.el (Bun 特定)
  
  ;;; -*- lexical-binding: t; -*-
  
  ;; TS 缩进 (Doom 默认 2, 不用改)
  (after! typescript-mode
    (setq typescript-indent-level 2))
  
  ;; lsp-mode TS 配置
  (after! lsp-mode
    ;; 用 vtsls (如果装了)
    (setq lsp-clients-typescript-server-args '("--stdio"))
  
    ;; 保存时 format + organize imports (通过 lsp,不用 format-all)
    (add-hook 'typescript-mode-hook
              (lambda ()
                (add-hook 'before-save-hook
                          (lambda ()
                            (call-interactively #'lsp-format-buffer)
                            (call-interactively #'lsp-organize-imports))
                          nil t))))
  
  ;; eglot 版本 (如果你确实想用 eglot,需要 hack languageId):
  (after! eglot
    (add-to-list 'eglot-server-programs
                 `((tsx-ts-mode typescript-ts-mode typescript-mode)
                   . ("typescript-language-server" "--stdio"
                      :initializationOptions
                      . ,(list :hostInfo "Doom Emacs")))))
  
  ;; Bun 运行命令
  (def-package! bun-run
    :unless (featurep! :lang javascript)
    :commands (my/bun-run my/bun-test))
  
  (defun my/bun-run ()
    "Run bun run dev in project root."
    (interactive)
    (let ((default-directory (or (projectile-project-root) default-directory)))
      (compile "bun run dev")))
  
  (defun my/bun-test ()
    "Run bun test in project root."
    (interactive)
    (let ((default-directory (or (projectile-project-root) default-directory)))
      (compile "bun test")))
  
  (map! :after typescript-mode
        :map typescript-mode-map
        :localleader
        "r" #'my/bun-run
        "t" #'my/bun-test
        "bf" #'lsp-format-buffer)  ; buffer format
  
  .dir-locals.el (项目级)
  
  放项目根:
  
  ((typescript-mode
    . ((eval . (setq-local compile-command "bun run dev"))
       (lsp-enabled . t)))
   (typescript-tsx-mode
    . ((eval . (setq-local compile-command "bun run dev"))
       (lsp-enabled . t))))
  
  Bun 1.3 (2026 年 2 月发布) 特性
  
  Bun 1.3 (https://tech-insider.org/bun-javascript-tutorial-rest-api-2026/) 让 .ts 文件直接跑,不用 tsconfig 也不用 transpile。所以你的 workflow:
  
  bun run src/index.ts        # 直接跑 TS
  bun test                    # 跑测试
  bun build src/index.ts --outdir dist  # 打包
  
  在 Emacs 里用 M-x compile 跑这些。
  
  TSX 额外注意
  
  Doom 的 TSX 处理 一直是问题点 (https://merrick.luois.me/posts/better-tsx-support-in-doom-emacs)。加 +tree-sitter flag 后用 typescript-ts-mode (tree-sitter
  版本),TSX 高亮才准。但 lsp-mode 检测 tsx 文件可能要 手动指 mode (https://github.com/doomemacs/doomemacs/blob/master/modules/lang/javascript/config.el):
  
  (after! typescript-mode
    (add-to-list 'auto-mode-alist '("\\.tsx\\'" . tsx-ts-mode)))
  
  我之前对的地方
  
  - LSP server 装 typescript-language-server (或 vtsls)
  - @types/bun 装上获得 Bun API 类型
  - M-. / M-? / M-, 键位
  - eshell 直接跑 bun 命令
  
  总结 (2026 最佳实践)
  
  1. Module: (javascript +typescript +tree-sitter +lsp) 在 :lang
  2. LSP mode: 用 lsp-mode,不用 eglot (TSX 有 bug)
  3. LSP server: vtsls (新项目) 或 typescript-language-server (老项目)
  4. Format: Doom (format +onsave) 模块 + Biome (lint+format)
  5. TSX: 加 +tree-sitter flag
  6. Bun: 直接 bun run file.ts,Emacs 里用 compile 命令
  
  ---
  Sources:
  - Doom Emacs JavaScript module README (https://github.com/doomemacs/doomemacs/blob/master/modules/lang/javascript/README.org)
  - Doom Emacs JavaScript config.el (https://github.com/doomemacs/doomemacs/blob/master/modules/lang/javascript/config.el)
  - Better TSX Support with Tree-sitter in Doom Emacs (Merrick Luo) (https://merrick.luois.me/posts/better-tsx-support-in-doom-emacs)
  - Modern Emacs TypeScript Web (React) Config with lsp-mode (Ovis Toica, 2024) (https://www.ovistoica.com/blog/2024-7-05-modern-emacs-typescript-web-tsx-config)
  - Doom Emacs LSP module (eglot vs lsp-mode) (https://github.com/doomemacs/doomemacs/blob/master/modules/tools/lsp/README.org)
  - Eglot TypeScript/JSX issue #624 (https://github.com/joaotavora/eglot/issues/624)
  - vtsls.com (https://vtsls.com/)
  - The Case for Bun in 2026 (Medium) (https://medium.com/@oliveryasuna.main/the-case-for-bun-in-2026-where-it-works-and-where-it-doesnt-1cf61a55d1c9)
  - Bun JavaScript Tutorial 2026 (Bun 1.3) (https://tech-insider.org/bun-javascript-tutorial-rest-api-2026/)
  - Doom Discourse: Set up LSP-mode or Eglot (https://discourse.doomemacs.org/t/set-up-lsp-mode-or-eglot-for-insert-language-here/62)
