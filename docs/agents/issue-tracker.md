# Issue tracker: GitHub

Issues และ spec ของโปรเจกต์นี้อยู่ใน GitHub Issues ใช้ `gh` CLI สำหรับทุก operation
ภาษาของเนื้อหาใน issue คือ **ภาษาไทย**

## ข้อกำหนดเบื้องต้น

ก่อนใช้งาน tracker ได้ ต้องมีสองอย่างนี้ก่อน (ยังไม่ได้ตั้งค่าใน repo นี้ ณ ตอนที่เขียนไฟล์นี้):

1. **ติดตั้ง `gh` CLI** และ authenticate (`gh auth login`)
2. **มี git remote ที่ชี้ไปยัง repo บน GitHub** — ปัจจุบัน repo นี้ยังไม่มี remote
   `gh` จะอ่าน repo จาก `git remote -v` ให้อัตโนมัติเมื่อรันภายใน clone
   ถ้ายังไม่มี remote ให้ระบุชัดเจนทุกครั้งด้วย `--repo <owner>/<repo>`

## Conventions

- **สร้าง issue**: `gh issue create --title "..." --body "..."` ใช้ heredoc สำหรับ body ที่หลายบรรทัด
- **อ่าน issue**: `gh issue view <number> --comments` พร้อม filter comment ด้วย `jq` และดึง label มาด้วย
- **รายการ issue**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` พร้อม `--label` และ `--state` ที่เหมาะสม
- **คอมเมนต์บน issue**: `gh issue comment <number> --body "..."`
- **เพิ่ม / ลบ label**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **ปิด issue**: `gh issue close <number> --comment "..."`

## Pull requests as a triage surface

**PRs as a request surface: no.**

_(เปลี่ยนเป็น `yes` ได้ถ้า repo นี้ถือว่า PR ภายนอกคือ feature request; `/triage` จะอ่านค่า flag นี้)_

เมื่อตั้งเป็น `yes` PR จะเดินผ่าน label และ state ชุดเดียวกับ issue โดยใช้คำสั่ง `gh pr` แทน:

- **อ่าน PR**: `gh pr view <number> --comments` และ `gh pr diff <number>` สำหรับ diff
- **รายการ PR ภายนอกเพื่อ triage**: `gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments` แล้วเก็บเฉพาะ `authorAssociation` เป็น `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR` หรือ `NONE` (ตัด `OWNER`/`MEMBER`/`COLLABORATOR` ทิ้ง)
- **คอมเมนต์ / label / ปิด**: `gh pr comment`, `gh pr edit --add-label`/`--remove-label`, `gh pr close`

GitHub ใช้ number space เดียวกันระหว่าง issue กับ PR ดังนั้น `#42` เปล่าๆ อาจเป็นได้ทั้งสองอย่าง: แก้ด้วย `gh pr view 42` แล้ว fallback ไป `gh issue view 42`

## เมื่อ skill พูดว่า "publish to the issue tracker"

สร้าง GitHub issue หนึ่งอัน

## เมื่อ skill พูดว่า "fetch the relevant ticket"

รัน `gh issue view <number> --comments`

## Wayfinding operations

ใช้โดย `/wayfinder` — **map** คือ issue เดียวที่มี **child issues** เป็น tickets

- **Map**: issue เดียวที่ติด label `wayfinder:map` เก็บเนื้อหา Notes / Decisions-so-far / Fog
  สร้างด้วย `gh issue create --label wayfinder:map`
- **Child ticket**: issue ที่ผูกกับ map ในรูปแบบ GitHub sub-issue (`gh api` ที่ sub-issues endpoint)
  ถ้า sub-issues เปิดใช้ไม่ได้ ให้เพิ่ม child ลงใน task list ใน body ของ map และใส่ `Part of #<map>` ไว้บนสุดของ body ของ child
  Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`)
  เมื่อถูก claim แล้ว ให้ assign ticket ไปที่ dev ที่เป็นผู้ขับเคลื่อน
- **Blocking**: ใช้ **native issue dependencies** ของ GitHub เป็นรูปแบบหลัก (มองเห็นได้ใน UI)
  เพิ่ม edge ด้วย `gh api --method POST repos/<owner>/<repo>/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`
  โดย `<blocker-db-id>` คือ **database id** ตัวเลขของ blocker (`gh api repos/<owner>/<repo>/issues/<n> --jq .id` — ไม่ใช่ `#number` และไม่ใช่ `node_id`)
  GitHub รายงานผลที่ `issue_dependencies_summary.blocked_by` (เฉพาะ blocker ที่ยังเปิดอยู่ คือ gate ที่ใช้งานจริง)
  ถ้า dependencies ใช้ไม่ได้ ให้ fallback เป็นบรรทัด `Blocked by: #<n>, #<n>` ที่บนสุดของ body ของ child
  Ticket จะถือว่า unblocked เมื่อ blocker ทุกตัวถูกปิดแล้ว
- **Frontier query**: รายการ child ที่ยังเปิดอยู่ของ map (`gh issue list --state open` จำกัดขอบเขตที่ sub-issues / task list ของ map)
  ตัดตัวที่มี blocker ที่ยังเปิดอยู่ (`issue_dependencies_summary.blocked_by > 0` หรือมี issue ที่เปิดอยู่ในบรรทัด `Blocked by`) หรือมี assignee ออก
  ตัวแรกตามลำดับใน map ชนะ
- **Claim**: `gh issue edit <n> --add-assignee @me` — write แรกของ session
- **Resolve**: `gh issue comment <n> --body "<answer>"` แล้ว `gh issue close <n>`
  แล้วเติม context pointer (gist + link) ลงในส่วน Decisions-so-far ของ map
