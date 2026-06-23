2010607

2010493


INSERT INTO `conversation_reports` (`id`, `account_id`, `conversation_id`, `department_id`, `visitor_id`, `is_answered`, `is_missed`, `is_abandoned`, `reopened`, `from_website`, `from_hub`, `user_ids`, `user_durations`, `chat_bot_durations`, `durations`, `in_queue_duration`, `answered_duration`, `total_duration`, `message_types`, `hub_provider`, `channel_line_id`, `widget_id`, `during_working_hours`, `extra_attributes`, `opened_at`, `answered_at`, `closed_at`, `created_at`, `updated_at`) VALUES
(2010493, 1067, 1832349, '04ca3250-7ce3-4ba0-9acf-b5469b2ba958', 956038, 1, 0, NULL, 0, 1, 0, '[1883]', '{\"1883\": 187}', NULL, NULL, 1, 187, 188, 'text', NULL, NULL, 'e6b7026a-2d6e-47cd-a56f-b37f348f859f', NULL, NULL, '2022-02-01 18:13:46', '2022-02-01 18:13:47', '2022-02-01 18:16:54', '2022-02-01 18:16:54', '2022-02-01 18:16:54'),
(2010607, 1067, 1832442, '04ca3250-7ce3-4ba0-9acf-b5469b2ba958', 956038, 1, 0, NULL, 0, 1, 0, '[1880]', '{\"1880\": 252}', NULL, NULL, 90, 252, 342, 'text', NULL, NULL, 'e6b7026a-2d6e-47cd-a56f-b37f348f859f', NULL, NULL, '2022-02-01 18:28:55', '2022-02-01 18:30:25', '2022-02-01 18:34:37', '2022-02-01 18:34:37', '2022-02-01 18:34:37');



SELECT * FROM conversation_reports WHERE channel_line_id is null ORDER by id DESC;


SELECT * FROM conversations WHERE id=9938405;

#----


UPDATE conversation_reports cr
JOIN channel_lines cl
  ON cl.account_id = cr.account_id
  AND cl.channel_id = 1
  AND cl.identifier = cr.widget_id
SET cr.channel_line_id = cl.id
WHERE cr.id IN (
  SELECT id FROM (
    SELECT id
    FROM conversation_reports
    WHERE widget_id IS NOT NULL
      AND channel_line_id IS NULL
    ORDER BY id
    LIMIT 100000
  ) t
);


SELECT COUNT(*) FROM conversation_reports
WHERE widget_id IS NOT NULL AND channel_line_id IS NULL;

SELECT * FROM conversation_reports
WHERE widget_id IS NOT NULL AND channel_line_id IS NULL;  



SELECT * FROM conversation_reports where id < 2010493 and channel_line_id is NULL and from_hub=0 ORDER by id desc limit 5000;

#---- 8753981


------------------------
HUB
6	instagram	NULL	2025-03-07 11:48:33	2025-03-07 11:48:33
5	email	NULL	2023-12-21 04:39:07	2023-12-21 04:39:07
4	whatsapp	NULL	2020-06-01 04:39:07	2020-06-01 04:39:07
3	viber	NULL	2020-06-01 04:39:07	2020-06-01 04:39:07
2	telegram	NULL	2020-06-01 04:39:07	2020-06-01 04:39:07
1	messenger	NULL	2020-06-01 04:39:07	2020-06-01 04:39:07



7	email	NULL
6	instagram	NULL
5	whatsapp	NULL
4	viber	NULL
3	telegram	NULL
2	messenger	NULL
1	website	NULL


სავარაუდოდ ესაა საბოლოო
SELECT
  cr.id                AS report_id,
  cr.conversation_id,
  cr.account_id        AS report_account_id,
  c.account_id         AS conv_account_id,
  c.extra_attributes ->> '$.social_hub.account.id' AS sh_account_id,
  a.identifier         AS page_identifier,
  cl.id                AS new_channel_line_id,
  cr.channel_line_id   AS current_value
FROM `livecaller.io`.conversation_reports cr
JOIN `livecaller.io`.conversations c
  ON c.id = cr.conversation_id
JOIN `hub.livecaller.io`.accounts a
  ON a.id = CAST(c.extra_attributes ->> '$.social_hub.account.id' AS UNSIGNED)
JOIN `livecaller.io`.channel_lines cl
  ON cl.channel_id = 2
  AND cl.account_id = c.account_id
  AND cl.identifier = a.identifier
WHERE cr.id IN (
  SELECT id FROM (
    SELECT cr2.id
    FROM `livecaller.io`.conversation_reports cr2
    JOIN `livecaller.io`.conversations c2 ON c2.id = cr2.conversation_id
    WHERE cr2.channel_line_id IS NULL
      AND c2.extra_attributes ->> '$.social_hub.account.id' IS NOT NULL
    ORDER BY cr2.id
    LIMIT 10000
  ) t
);



id	identifier	name	lc_account_id
795	prod@livecaller.email	prod@livecaller.email	176
1085	superapp@tnet.ge	superapp@tnet.ge	4728
1086	1117857098073654	Logisticsge	176
1087	8762497261	integrations test	176
1088	info@myhome.ge	info@myhome.ge	2591
1089	7903178474	BLD_ Bot	2642
1092	840718305789835	Crocoshop.ge	4646
1093	1159618767404496	Cosmo.ge	4646

ესაა ბოლო: 986957590737063937
5464 + 8 = 5472


32591	87	32690	f6195460-d727-421e-860e-9dba9149bb1a	67732	1	0	NULL	0	0	1	[165]	"{""165"": 3716}"	NULL	NULL	97	3716	3813	text	messenger	NULL	NULL	NULL	NULL	2020-06-01 13:36:29	2020-06-01 13:38:06	2020-06-01 14:40:02	2020-10-28 01:57:36	2020-10-28 01:57:36
33865	1	32602	9caa8250-e722-4350-9b46-89eb639e27fa	67610	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	435394	NULL	435394	text	messenger	NULL	NULL	NULL	NULL	2020-06-01 05:43:27	NULL	2020-06-06 06:40:01	2020-10-28 01:57:36	2020-10-28 01:57:36
35469	1	35596	9caa8250-e722-4350-9b46-89eb639e27fa	72597	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	2391	NULL	2391	text	messenger	NULL	NULL	NULL	NULL	2020-06-12 01:10:11	NULL	2020-06-12 01:50:02	2020-10-28 01:57:36	2020-10-28 01:57:36
35473	1	35600	9caa8250-e722-4350-9b46-89eb639e27fa	72609	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	1970	NULL	1970	text	messenger	NULL	NULL	NULL	NULL	2020-06-12 04:37:11	NULL	2020-06-12 05:10:01	2020-10-28 01:57:36	2020-10-28 01:57:36
36316	96	36441	e9a34377-dddf-45c4-971b-25abd31d1620	74048	1	0	NULL	0	0	1	[185]	"{""185"": 4125}"	NULL	NULL	113	4125	4238	text	messenger	NULL	NULL	NULL	NULL	2020-06-15 23:49:23	2020-06-15 23:51:16	2020-06-16 01:00:01	2020-10-28 01:57:36	2020-10-28 01:57:36
36674	24	36803	825a528a-7972-451d-a747-65f7effddf91	74652	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	2118	NULL	2118	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 13:24:44	NULL	2020-06-17 14:00:02	2020-10-28 01:57:36	2020-10-28 01:57:36
36675	24	36801	825a528a-7972-451d-a747-65f7effddf91	74648	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	2262	NULL	2262	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 13:22:20	NULL	2020-06-17 14:00:02	2020-10-28 01:57:36	2020-10-28 01:57:36
36683	24	36811	825a528a-7972-451d-a747-65f7effddf91	74678	1	0	NULL	0	0	1	[170]	"{""170"": 391}"	NULL	NULL	1463	391	1854	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 13:46:51	2020-06-17 14:11:14	2020-06-17 14:17:45	2020-10-28 01:57:36	2020-10-28 01:57:36
36696	24	36820	825a528a-7972-451d-a747-65f7effddf91	74696	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	1801	NULL	1801	text	viber	NULL	NULL	NULL	NULL	2020-06-17 14:10:01	NULL	2020-06-17 14:40:02	2020-10-28 01:57:36	2020-10-28 01:57:36
36697	24	36830	825a528a-7972-451d-a747-65f7effddf91	74705	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	1959	NULL	1959	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 14:17:22	NULL	2020-06-17 14:50:01	2020-10-28 01:57:36	2020-10-28 01:57:36
36699	24	36821	825a528a-7972-451d-a747-65f7effddf91	74697	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	2400	NULL	2400	text	viber	NULL	NULL	NULL	NULL	2020-06-17 14:10:01	NULL	2020-06-17 14:50:01	2020-10-28 01:57:36	2020-10-28 01:57:36
36707	24	36840	825a528a-7972-451d-a747-65f7effddf91	74718	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	2054	NULL	2054	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 14:25:47	NULL	2020-06-17 15:00:01	2020-10-28 01:57:36	2020-10-28 01:57:36
36711	24	36853	825a528a-7972-451d-a747-65f7effddf91	74737	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	1949	NULL	1949	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 14:47:33	NULL	2020-06-17 15:20:02	2020-10-28 01:57:36	2020-10-28 01:57:36
36712	24	36851	825a528a-7972-451d-a747-65f7effddf91	74735	0	1	NULL	0	0	1	NULL	NULL	NULL	NULL	2030	NULL	2030	text	messenger	NULL	NULL	NULL	NULL	2020-06-17 14:46:12	NULL	2020-06-17 15:20:02	2020-10-28 01:57:36	2020-10-28 01:57:36