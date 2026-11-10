---
spec_id: 001-persistent-tasks-and-web-task-board
title: Persistent tasks and web task board
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task board

## Problem

The taskboard10 task API keeps tasks in its own memory: every restart or redeploy loses all tasks, and each running instance of the API keeps its own separate list, so two calls can disagree about which tasks exist. People also have no way to use the board except by calling the API by hand, because the taskboard10 web front end does not show or create tasks yet. Until tasks survive a restart and can be seen and added in a browser, the system cannot be used or demonstrated by anyone but a developer.

## Stories

As a person using the task board, I want the tasks I created to still be there after the system restarts or is redeployed, so that I can rely on the board as my record.
**Repos:** taskboard10-api, taskboard10-gitops

As a person using the task board, I want to open the board in my browser and see all tasks with their due dates, so that I know what is to be done.
**Repos:** taskboard10-web, taskboard10-api

As a person using the task board, I want to add a task with a title and, if I like, a due date from the browser, so that I do not need to call the API by hand.
**Repos:** taskboard10-web, taskboard10-api

## Acceptance criteria

- **AC-001.1** Given tasks "A" and then "B" were created, when every running instance of the task API is restarted or replaced by a new version, then `GET /tasks` and the web board both still show "A" before "B", each with the same `id`, `title` and `due_date` as before the restart.
- **AC-001.2** Given the task API runs with 2 instances and a task was just created through `POST /tasks`, when a client sends `GET /tasks` 10 times in a row, then every one of the 10 responses contains that task.
- **AC-001.3** Given tasks exist, when a person opens the web board's start page, then it lists every task, oldest first, each showing its title and its due date as `YYYY-MM-DD`, or the text "No due date" when it has none.
- **AC-001.4** Given no task exists, when a person opens the web board's start page, then it shows the text "No tasks yet" and the form to add a task.
- **AC-001.5** Given the web board is open, when a person enters the title "Buy milk", picks the due date 2026-12-01 (or leaves it empty) and submits the form, then the task appears at the end of the list without a manual page reload, the form is emptied, and `GET /tasks` on the task API returns the same task.
- **AC-001.6** Given the web board is open, when a person submits the form with an empty or whitespace-only title, or a title longer than 200 characters, then the board shows a message next to the title field ("Enter a title" or "Title must be at most 200 characters"), keeps the entered values, and no task is created.
- **AC-001.7** Given the database is unreachable, when a client sends `GET /tasks` or `POST /tasks`, then the task API answers 503 with a `detail` saying the task store is unavailable, and when a person opens the web board, then it shows "Tasks could not be loaded. Try again later." instead of an empty list.
- **AC-001.8** Given the web board is open and the task API or its database becomes unreachable, when a person submits a valid task, then the board shows "Task could not be saved. Try again." and keeps the entered title and due date in the form.

## Non-goals

- Changing the task model or the API contract of taskboard10-api spec 001 (fields `id`, `title`, `due_date`; endpoints `POST /tasks` and `GET /tasks`; validation rules and 422 answers stay as they are).
- Editing, completing or deleting tasks, in the API or on the board.
- Filtering, sorting options, search and pagination.
- Users, sign-in, access control and per-user boards: anyone who can reach the web board may see all tasks and add tasks, as with the task API; access control is a later spec.
- Carrying over tasks that exist in the API's memory at the moment of the switch.
- Database backups and restore; running the database or the board in staging and prod (they start in dev like every new component).

## Assumptions

- **A1** Tasks held in the API's memory when the persistent store goes live are not carried over; the board starts empty (other option: export and import them once; affects AC-001.1, AC-001.4).
- **A2** The board is the start page of the web front end; it lists and adds tasks on one page (other option: separate list and "new task" pages; affects AC-001.3 to AC-001.6).
- **A3** The board shows all tasks oldest first, matching the API's order, with due dates as `YYYY-MM-DD` and "No due date" for none (other option: newest first, a localized date format; affects AC-001.3).
- **A4** The board checks the title rules (required, at most 200 characters after trimming) before sending, with the messages "Enter a title" and "Title must be at most 200 characters"; the due date is picked with a date field, so a malformed date cannot be entered (other option: rely on the API's 422 answer only; affects AC-001.6).
- **A5** When the database is unreachable the API answers 503 with a `detail`, and the board shows "Tasks could not be loaded. Try again later." or "Task could not be saved. Try again." (other option: 500, other wording; affects AC-001.7, AC-001.8).
- **A6** A new task appears on the board right after it is saved, without a reload; tasks created by other people or scripts appear on the next page load, not live (other option: live updates; affects AC-001.5).
- **A7** The database is a Postgres database run inside the cluster by the platform's database operator, declared as a resource of the system, with no paid external service (other option: a managed cloud database; affects AC-001.1, AC-001.2).

## Open questions

- ~~May anyone who can reach the web board see all tasks and add tasks without signing in, as the task API already allows (taskboard10-api spec 001)?~~ Answered 2026-11-10: yes, no sign-in in this spec; access control is a later spec, see Non-goals.

## Changelog

- 2026-11-10 created
- 2026-11-10 amended: open question on sign-in answered (none in this spec, anyone who reaches the board may see and add tasks), recorded under Non-goals; criteria unchanged
