Event stuck in "Processing" (sync_status / cron_status = InProgress)
This doc lists why an event can remain InProgress in the DB (Supabase/PostgreSQL) on production while it completes on staging, and where both backend and Flutter touch this status.

1. Where status is stored and used
Backend (Getin-Scanner-be)
Schema (Prisma): events.sync_status (enum SyncStatus: InProgress | Completed), events.cron_status (enum CronStatus: InProgress | Completed).
Default: New rows use sync_status: InProgress, cron_status: Completed (see prisma/schema.prisma).
Who sets InProgress: Manual sync sets sync_status; cron sets cron_status before processing an event.
Who sets Completed: Inside fetchAndUpsertAttendeesBatchWise (success or catch), and in updateEventStatusAfterProcessing (bulk per user). Safeguard: A finally in markeUsersProcessing now also sets both statuses to Completed so a single event never stays stuck after an error.
Flutter (GetIn-ScannerApp)
Source of truth: PowerSync/Supabase syncs the events table; Flutter reads syncStatus / cronStatus from the synced events row.
Where it’s used:
lib/backend/supabase/database/tables/events.dart – syncStatus, cronStatus (strings from DB).
lib/components/event_card/event_card_widget.dart (e.g. ~116) – If event.syncStatus == 'InProgress' (or priority sync state), shows “syncing” / disabled state.
lib/components/event_selection_bottom_sheet/event_selection_bottom_sheet_widget.dart (e.g. ~305, 431) – Filters or treats events with syncStatus == 'InProgress' && priority == 1 differently.
lib/components/event_card_for_scanner/event_card_for_scanner_widget.dart (e.g. ~147) – Disables or adjusts UI when event?.syncStatus == 'InProgress'.
So if the DB row stays InProgress, the Flutter app will keep showing that event as “processing” until the backend sets it to Completed.

2. Cases where status can remain InProgress (before safeguard)
These are the situations that could leave an event stuck before the new finally safeguard. Some can still matter if the safeguard fails (e.g. DB error in finally) or in edge cases.

#	Case	Where (backend)	Why it stays InProgress
1	Error inside fetchAndUpsertAttendeesBatchWise	events.controller.ts – fetchAndUpsertAttendeesBatchWise	Status set to InProgress before the call; if fetchAndUpsertAttendeesBatchWise throws (e.g. participants API failure, DB error) before its internal catch runs, we used to exit without setting Completed. Mitigation: finally in markeUsersProcessing now sets Completed for that event.
2	Participants API fails	fetchAndUpsertAttendeesBatchWise – axios.post(apiUrl, …)	If the Get-in participants request throws (network, 5xx), we never reach the “no participants” update or the catch that sets Completed. Mitigation: same finally sets Completed.
3	Process killed / crash	Anywhere after setting InProgress	Server restart, OOM, or kill after we set an event to InProgress but before we set it to Completed. Mitigation: next successful run calls updateEventStatusAfterProcessing, which bulk-sets all that user’s InProgress events to Completed. If the process never runs again for that user, event can stay stuck until a manual fix or a cleanup job.
4	Cron runs but processUserData returns 0 users	fetchEventsForUsers → processUserData({})	Cron calls with no user_id; processUserData uses prisma.$replica().creators.findMany(). If replica is down or returns [], we never enter the for (const user of users) loop, so we never call updateEventStatusAfterProcessing. Any event left InProgress from a previous run stays stuck. More likely in prod if replica URL or connectivity differs.
5	Event no longer “live”	markeUsersProcessing – liveEvents query	We only process events with end_date > now. If an event was set InProgress and then its end_date passed (or timezone/clock differs in prod), the next run won’t include it in liveEvents, so we never run attendee sync for it and previously we didn’t set Completed. Mitigation: updateEventStatusAfterProcessing sets all that user’s InProgress events to Completed after processing, so once we process that user again, we clear it. If that user is never processed again (e.g. 0 users in cron), event can stay stuck.
6	New event created with InProgress, not in live list	processEventData	New or updated event is written with sync_status: InProgress (manual sync). If it’s not in liveEvents (e.g. end_date in the past), we never call fetchAndUpsertAttendeesBatchWise for it. Mitigation: same as #5 – updateEventStatusAfterProcessing clears it when that user is processed.
7	Replica lag (read-your-writes)	Backend uses prisma (primary) and prisma.$replica()	Writes go to primary; some reads use replica. If Flutter/PowerSync reads from a replica that lags, it can still see InProgress after the backend has set Completed. Staging vs prod: different replica lag or different DB topology can make this visible only in prod.
8	Schema default	prisma/schema.prisma	sync_status defaults to InProgress. Any insert that doesn’t set sync_status explicitly will create an InProgress row. All current code paths that create/update events set status explicitly; this is a general reminder.
9	Multiple managers (manual sync)	Skip condition in markeUsersProcessing	Staging: one manager; production: multiple. Before fix: Manual sync only skipped when cron was in progress, not when sync_status === InProgress. So Manager A completes → Completed; Manager B starts sync, sets InProgress again; if B fails, event stuck. Fix: We now skip manual sync when sync_status === InProgress too.
3. Staging vs production – why prod might stay “processing”
Multiple managers: Staging often has one manager; production has many. A second manager's manual sync could previously set an event back to InProgress after the first completed; if that second run failed, the event stayed stuck. Fix: manual sync now skips when sync_status === InProgress (case #9 above).
Cron / processUserData: In prod, cron might hit a different DB (replica) or env; if processUserData({}) returns [] (e.g. replica failure, no DATABASE_URL/replica URL), no user loop runs → no updateEventStatusAfterProcessing → stuck events never cleared.
Replica lag: Prod may have more read replicas or longer lag; Flutter may read from a replica that hasn’t received the Completed update yet.
Errors in prod only: Network/timeouts to Get-in API, or DB errors in prod, can cause fetchAndUpsertAttendeesBatchWise to throw. The new finally safeguard should still set Completed; if the finally update fails (e.g. DB error), the event can remain InProgress.
Multiple instances: If more than one backend instance runs cron, one can set an event to InProgress and another might skip it (already InProgress), so the second never clears it; updateEventStatusAfterProcessing at the end of each user’s run should still clear that user’s events.
4. Rule: never overwrite Completed → InProgress
To prevent the bug where any function could set an already-Completed event back to InProgress:

In the sync loop (markeUsersProcessing): Before setting an event to InProgress, we skip it if both sync_status and cron_status are already Completed. So we never overwrite Completed → InProgress there.
In processEventData (update path): When updating an existing event on manual sync, we set sync_status to InProgress only if it is not already Completed; if it is Completed we leave it Completed.
So once an event is Completed, no code path will change it back to InProgress.

5. Backend safeguard added (finally block)
In src/controller/events.controller.ts, in the loop over liveEvents, the block that calls fetchAndUpsertAttendeesBatchWise now has a finally that:

Calls markEventAsDone(liveEvent.event_id) (memory lock).
Updates the event in the DB: sync_status: Completed, cron_status: Completed.
So even if fetchAndUpsertAttendeesBatchWise throws, that event is no longer left InProgress. If the update in finally fails, we log and the event can still stay InProgress (same as case #1 with a failed safeguard).

6. What to do for already-stuck events
One-off fix in DB (Supabase):

Run:
UPDATE events SET sync_status = 'Completed', cron_status = 'Completed' WHERE sync_status = 'InProgress' OR cron_status = 'InProgress';
Or restrict by event_id / creator_user if you want to be selective.
Optional: cleanup job in backend
A small cron or endpoint that periodically sets to Completed any events that have been InProgress for longer than e.g. 1 hour (with a safety check so you don’t overwrite a currently running sync if you add a “last_updated” or similar).

Flutter
No code change needed for “where” status is used; once the DB has Completed, the next PowerSync sync will update the app and the event will no longer show as processing.

7. Quick reference – files
Layer	File	What
Backend	prisma/schema.prisma	events.sync_status, events.cron_status enums and defaults
Backend	src/controller/events.controller.ts	All InProgress/Completed updates; markeUsersProcessing, fetchAndUpsertAttendeesBatchWise, updateEventStatusAfterProcessing, and the new finally safeguard
Flutter	lib/backend/supabase/database/tables/events.dart	syncStatus, cronStatus from DB
Flutter	lib/components/event_card/event_card_widget.dart	Shows “syncing” when syncStatus == 'InProgress'
Flutter	lib/components/event_selection_bottom_sheet/event_selection_bottom_sheet_widget.dart	Filter/behavior for syncStatus == 'InProgress' && priority == 1
Flutter	lib/components/event_card_for_scanner/event_card_for_scanner_widget.dart	Behavior when syncStatus == 'InProgress'
