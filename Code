**
 * LAS COMMAND PORTAL — Server-side Apps Script
 * ---------------------------------------------
 * Backed by a Google Sheet ("Users" tab) and the public Roblox APIs.
 *
 * SETUP:
 * 1. Create a Google Sheet, then Extensions > Apps Script.
 * 2. Paste this file as Code.gs, and Index.html as a separate HTML file.
 * 3. Deploy > New deployment > Web app.
 *    - Execute as: Me
 *    - Who has access: Anyone within your organisation (or "Anyone" if you want it public)
 * 4. Open the deployed URL.
 *
 * The sheet is created automatically on first run — no manual setup needed
 * beyond having an empty spreadsheet the script is bound to.
 */

// ---------- CONFIG ----------

const MAIN_GROUP_ID = 860308753;      // UK London Ambulance Serv-ce
const CRITICAL_CARE_GROUP_ID = 637235433; // LAS Critical Care Team
const OMBUDSMAN_GROUP_ID = 882881950; // LAS Ombudsman — independent membership track, see below
const EMAIL_DOMAIN = 'las.wuk.sg';
const SESSION_TTL_SECONDS = 6 * 60 * 60; // 6 hours
const USERS_SHEET_NAME = 'Users';

// Rank table. `match` strings are lower-cased substrings checked against the
// member's Roblox role name in the CRITICAL CARE TEAM group to auto-detect
// their rank.
//
// `reportLevel` is a SEPARATE, finer-grained scale used only by the Weekly
// Reports review chain (TM < OM < GM < AD < DIR/DCE/CEO), since TM and OM
// share a general access `level` (Bronze), as do GM and AD (Silver), but
// they are NOT equal for report review/signoff purposes. Ranks with no
// reportLevel don't submit/review weekly reports at all.
const RANKS = [
  { id: 'jrp',  label: 'Junior Paramedic', tier: 'Probationary Servicemen',              level: 1, match: ['junior paramedic'] },
  { id: 'para', label: 'Paramedic',        tier: 'Probationary Servicemen',              level: 1, match: ['paramedic'] }, // checked after junior paramedic
  { id: 'sp',   label: 'Specialist Paramedic', tier: 'Servicemen',                       level: 2, match: ['specialist paramedic'] },
  { id: 'advp', label: 'Adv. Paramedic',   tier: 'Supervisory',                          level: 3, match: ['advanced paramedic', 'adv. paramedic', 'adv paramedic'] },
  { id: 'cons', label: 'Consultant',       tier: 'Supervisory',                          level: 3, match: ['consultant'] },
  { id: 'tm',   label: 'TM',               tier: 'Bronze (Division Command)',            level: 4, match: ['team manager', ' tm', 'tm '], reportLevel: 1 },
  { id: 'om',   label: 'OM',               tier: 'Bronze (Division Command)',            level: 4, match: ['operations manager', ' om', 'om '], reportLevel: 2 },
  { id: 'gm',   label: 'GM',               tier: 'Silver (Division Leadership)',         level: 5, match: ['general manager', ' gm', 'gm '], reportLevel: 3 },
  { id: 'ad',   label: 'AD',               tier: 'Silver (Division Leadership)',         level: 5, match: ['assistant director', ' ad', 'ad '], note: 'Division Lead', reportLevel: 4 },
  { id: 'dir',  label: 'DIR',              tier: 'Gold (Admin / Service Oversight)',     level: 6, match: ['director', ' dir'], note: 'Gold Staff', reportLevel: 5 },
  { id: 'dce',  label: 'DCE',              tier: 'Gold (Admin / Service Oversight)',     level: 6, match: ['deputy chief executive', 'dce'], reportLevel: 5 },
  { id: 'ceo',  label: 'CEO',              tier: 'Gold (Admin / Service Oversight)',     level: 6, match: ['chief executive officer', 'ceo'], reportLevel: 5 },
  { id: 'dev',  label: 'Dev',              tier: 'Project Management (Admin)',           level: 7, match: ['developer', ' dev'], reportLevel: 5 },
  { id: 'lead', label: 'Lead',             tier: 'Project Management (Admin)',           level: 7, match: ['lead'], reportLevel: 5 },

  // Ombudsman track — a completely separate membership path (see
  // checkRobloxEligibility) with no reportLevel (they don't take part in
  // the Weekly Reports chain) and no `match` (never auto-detected off a
  // CCT group role name, only an OMB group role name via
  // mapOmbRoleNameToRank_). `level` reuses the same 1-7 scale purely so
  // existing tab-gating math works, but OMB_TAB_WHITELIST below stops
  // them seeing operational tools that number alone would otherwise grant.
  { id: 'omb_inv',  label: 'OMB Investigator', tier: 'Ombudsman — Investigator (Bronze Command)',            level: 4, match: [], isOMB: true },
  { id: 'omb_lead', label: 'OMB Leadership',   tier: 'Ombudsman — Leadership (Silver Command, Audit)',       level: 5, match: [], isOMB: true }
];

// Weekly Reports review chain, keyed by reportLevel. `reviews` is the
// reportLevel this rank can review/signoff (null = none, 'all' = every
// level below). `viewScope` is which reportLevels this rank can view the
// history of, in addition to their own reports (always viewable).
const REPORT_CHAIN = {
  1: { reviews: null, viewBelow: [] },            // TM
  2: { reviews: 1,    viewBelow: [1] },           // OM — reviews TM, views TM
  3: { reviews: 2,    viewBelow: [1, 2] },        // GM — reviews OM, views OM+TM
  4: { reviews: 3,    viewBelow: [1, 2, 3] },     // AD — reviews GM, views all below
  5: { reviews: 'all', viewBelow: [1, 2, 3, 4] }  // DIR/DCE/CEO/Dev/Lead — reviews & views everything
};

function findRankById_(id) {
  return RANKS.filter(function (r) { return r.id === id; })[0] || null;
}

// Tabs on the dashboard and the minimum rank level required to open them.
const TABS = [
  { id: 'modules',      label: 'Module Logging',              minLevel: 3, note: 'SV+' },
  { id: 'incidents',    label: 'Incident Report Logging',     minLevel: 3, note: 'SV+' },
  { id: 'events',       label: 'Event Logging',                minLevel: 3, note: 'SV+' },
  { id: 'quota',        label: 'Quota Tracker',                minLevel: 2, note: 'Servicemen+' },
  { id: 'academy',      label: 'Academy Tracker',              minLevel: 1, note: 'All — view differs by rank' },
  { id: 'myDiscipline', label: 'My Discipline',                minLevel: 1, note: 'All — your own record only' },
  { id: 'weekly',       label: 'Weekly Reports',               minLevel: 4, note: 'Manager+, levels apply' },
  { id: 'activityDisc', label: 'Activity Disciplinary Dashboard', minLevel: 4, note: 'Manager+ / OMB Investigator+' },
  { id: 'internalDisc', label: 'Internal Disciplinary Dashboard', minLevel: 5, note: 'Silver Command+ / OMB Leadership' },
  { id: 'audit',        label: 'Audit Division',               minLevel: 5, note: 'Silver Command+ / OMB Leadership' }
];

// Ombudsman members only ever see this subset of tabs, regardless of their
// equivalent level — they're an oversight body, not operational CCT staff,
// so they don't get Module/Incident/Event Logging, Quota, or Academy access
// just because their rank number happens to be high enough.
const OMB_TAB_WHITELIST = ['myDiscipline', 'activityDisc', 'internalDisc', 'audit'];

// ---------- WEB APP ENTRY ----------

function doGet(e) {
  const template = HtmlService.createTemplateFromFile('Index');
  template.tabs = TABS;
  template.reportChain = REPORT_CHAIN;
  return template.evaluate()
    .setTitle('LAS Critical Care Team Portal')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}

// ---------- SHEET HELPERS ----------

function getUsersSheet_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(USERS_SHEET_NAME);
  if (!sheet) {
    sheet = ss.insertSheet(USERS_SHEET_NAME);
    sheet.appendRow([
      'Username', 'Email', 'PasswordHash', 'Salt',
      'RankId', 'RankLabel', 'RankTier', 'RankLevel',
      'RobloxUserId', 'CreatedAt'
    ]);
    sheet.setFrozenRows(1);
  }
  return sheet;
}

function findUserRow_(username) {
  const sheet = getUsersSheet_();
  const data = sheet.getDataRange().getValues();
  const uname = username.toLowerCase();
  for (let i = 1; i < data.length; i++) {
    if (String(data[i][0]).toLowerCase() === uname) {
      return { rowIndex: i + 1, row: data[i] };
    }
  }
  return null;
}

function rowToUser_(row) {
  const rank = findRankById_(row[4]);
  return {
    username: row[0],
    email: row[1],
    rankId: row[4],
    rankLabel: row[5],
    rankTier: row[6],
    rankLevel: Number(row[7]),
    reportLevel: rank && rank.reportLevel ? rank.reportLevel : 0,
    isOMB: !!(rank && rank.isOMB),
    robloxUserId: row[8]
  };
}

// ---------- PASSWORD HASHING ----------

function hashPassword_(password, salt) {
  const digest = Utilities.computeDigest(
    Utilities.DigestAlgorithm.SHA_256,
    salt + password,
    Utilities.Charset.UTF_8
  );
  return digest.map(function (b) {
    const v = (b < 0 ? b + 256 : b).toString(16);
    return v.length === 1 ? '0' + v : v;
  }).join('');
}

// ---------- ROBLOX API ----------

function robloxGetUserId_(username) {
  const resp = UrlFetchApp.fetch('https://users.roblox.com/v1/usernames/users', {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify({ usernames: [username], excludeBannedUsers: true }),
    muteHttpExceptions: true
  });
  if (resp.getResponseCode() !== 200) return null;
  const data = JSON.parse(resp.getContentText());
  if (!data.data || data.data.length === 0) return null;
  return data.data[0]; // { id, name, displayName }
}

function robloxGetGroupRoles_(userId) {
  const resp = UrlFetchApp.fetch(
    'https://groups.roblox.com/v1/users/' + userId + '/groups/roles',
    { muteHttpExceptions: true }
  );
  if (resp.getResponseCode() !== 200) return [];
  const data = JSON.parse(resp.getContentText());
  return data.data || []; // [{ group:{id,name}, role:{id,name,rank} }, ...]
}

function mapRoleNameToRank_(roleName) {
  const name = ' ' + roleName.toLowerCase() + ' ';
  let best = null;
  RANKS.forEach(function (rank) {
    for (let j = 0; j < rank.match.length; j++) {
      if (name.indexOf(rank.match[j]) !== -1) {
        if (!best || rank.level > best.level) best = rank;
        break;
      }
    }
  });
  return best;
}

// Ombudsman role names map to exactly two internal ranks: "Investigator"
// or Leadership (Executive/Chief). Anything else needs manual assignment,
// same fallback as the CCT track.
function mapOmbRoleNameToRank_(roleName) {
  const name = ' ' + roleName.toLowerCase() + ' ';
  if (name.indexOf('chief') !== -1 || name.indexOf('executive') !== -1) return findRankById_('omb_lead');
  if (name.indexOf('investigator') !== -1) return findRankById_('omb_inv');
  return null;
}

/**
 * Checks Roblox group membership + eligibility for a given username.
 * Returns { eligible, reason, robloxUserId, rank } — rank is null if the
 * member is in the relevant group(s) but their role name couldn't be
 * auto-mapped (a Manager should assign their rank manually in that case).
 *
 * The LAS Ombudsman group is a completely INDEPENDENT membership track:
 * being an Ombudsman member is sufficient on its own — it does NOT also
 * require membership of the two CCT groups. If someone is in the OMB
 * group, that's checked first and wins; only if they're not in it do we
 * fall through to the normal CCT-groups requirement.
 */
function checkRobloxEligibility(username) {
  username = (username || '').trim();
  if (!username) return { eligible: false, reason: 'Enter a Roblox username.' };

  const userInfo = robloxGetUserId_(username);
  if (!userInfo) {
    return { eligible: false, reason: 'No Roblox account found with that username.' };
  }

  const groupRoles = robloxGetGroupRoles_(userInfo.id);

  // ---- Ombudsman track ----
  const ombEntry = groupRoles.filter(function (g) { return g.group.id === OMBUDSMAN_GROUP_ID; })[0];
  if (ombEntry) {
    return {
      eligible: true,
      robloxUserId: userInfo.id,
      robloxUsername: userInfo.name,
      roleNameInGame: ombEntry.role.name,
      rank: mapOmbRoleNameToRank_(ombEntry.role.name) // may be null — see note above
    };
  }

  // ---- Standard CCT track ----
  const mainEntry = groupRoles.filter(function (g) { return g.group.id === MAIN_GROUP_ID; })[0];
  const ccEntry = groupRoles.filter(function (g) { return g.group.id === CRITICAL_CARE_GROUP_ID; })[0];

  if (!mainEntry) {
    return { eligible: false, reason: 'Not a member of the UK London Ambulance Serv-ce group, or the LAS Ombudsman group.', robloxUserId: userInfo.id };
  }
  if (!ccEntry) {
    return { eligible: false, reason: 'Not a member of the LAS Critical Care Team group (required for eligibility).', robloxUserId: userInfo.id };
  }

  const rank = mapRoleNameToRank_(ccEntry.role.name);
  return {
    eligible: true,
    robloxUserId: userInfo.id,
    robloxUsername: userInfo.name,
    roleNameInGame: ccEntry.role.name,
    rank: rank // may be null — see note above
  };
}

// ---------- ACCOUNT CREATION ----------

/**
 * Creates a new account. `overrideRankId` lets a Manager manually set the
 * rank when auto-detection from the Roblox role name fails.
 */
function createAccount(username, password, overrideRankId) {
  username = (username || '').trim();
  if (!username || !password) {
    return { success: false, message: 'Username and password are required.' };
  }
  if (password.length < 8) {
    return { success: false, message: 'Password must be at least 8 characters.' };
  }
  if (findUserRow_(username)) {
    return { success: false, message: 'An account already exists for that Roblox username.' };
  }

  const check = checkRobloxEligibility(username);
  if (!check.eligible) {
    return { success: false, message: check.reason };
  }

  let rank = check.rank;
  if (!rank && overrideRankId) {
    rank = RANKS.filter(function (r) { return r.id === overrideRankId; })[0] || null;
  }
  if (!rank) {
    return {
      success: false,
      message: 'Could not automatically match the in-game role "' + check.roleNameInGame +
        '" to a rank. Ask a Manager to create this account and manually select a rank.',
      needsManualRank: true,
      roleNameInGame: check.roleNameInGame
    };
  }

  const salt = Utilities.getUuid();
  const hash = hashPassword_(password, salt);
  const email = username + '@' + EMAIL_DOMAIN;

  getUsersSheet_().appendRow([
    username, email, hash, salt,
    rank.id, rank.label, rank.tier, rank.level,
    check.robloxUserId, new Date()
  ]);

  return { success: true, message: 'Account created for ' + username + ' (' + rank.label + ').' };
}

// ---------- LOGIN / SESSION ----------

function login(loginId, password) {
  const username = (loginId || '').split('@')[0].trim();
  const found = findUserRow_(username);
  if (!found) return { success: false, message: 'No account found for that username.' };

  const row = found.row;
  const salt = row[3];
  const storedHash = row[2];
  const attemptHash = hashPassword_(password, salt);
  if (attemptHash !== storedHash) {
    return { success: false, message: 'Incorrect password.' };
  }

  const user = rowToUser_(row);
  const token = Utilities.getUuid();
  CacheService.getScriptCache().put('session_' + token, JSON.stringify(user), SESSION_TTL_SECONDS);

  return { success: true, token: token, user: user, tabs: visibleTabsFor_(user) };
}

function sessionUser_(token) {
  if (!token) return null;
  const cached = CacheService.getScriptCache().get('session_' + token);
  if (!cached) return null;
  return JSON.parse(cached);
}

function getSession(token) {
  const user = sessionUser_(token);
  if (!user) return null;
  return { user: user, tabs: visibleTabsFor_(user) };
}

function logout(token) {
  if (token) CacheService.getScriptCache().remove('session_' + token);
  return { success: true };
}

function visibleTabsFor_(user) {
  return TABS.filter(function (t) {
    if (user.rankLevel < t.minLevel) return false;
    if (user.isOMB && OMB_TAB_WHITELIST.indexOf(t.id) === -1) return false;
    return true;
  }).map(function (t) { return t.id; });
}

function getRankOptions() {
  return RANKS.map(function (r) {
    return { id: r.id, label: r.label, tier: r.tier, note: r.note || '' };
  });
}

// =====================================================================
// EVENT LOGS
// =====================================================================
// Visibility: Manager+ see every event. Supervisory (level 3) only see
// events where they are the Host or the Co-Host — not other SVs' events.

const EVENT_LOG_SHEET_NAME = 'EventLogs';
const EVENT_TYPES = [
  'CCT Academy Training',
  'HEMS Qualification Course',
  'MIRU Academy Training',
  'Division Deployment',
  'Joint Deployment'
];

function getEventTypes() {
  return EVENT_TYPES;
}

function getEventLogSheet_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(EVENT_LOG_SHEET_NAME);
  if (!sheet) {
    sheet = ss.insertSheet(EVENT_LOG_SHEET_NAME);
    sheet.appendRow([
      'Id', 'CreatedAt', 'LoggedBy', 'TimeOfEvent', 'Host', 'CoHost', 'EventType',
      'AttendeesJson', 'PerformanceJson', 'WhatWentWell', 'ImprovementAreas',
      'IncidentLogId', 'AdditionalComments', 'ReviewedBy', 'ReviewedAt'
    ]);
    sheet.setFrozenRows(1);
  }
  return sheet;
}

function safeParseJson_(text, fallback) {
  try { return JSON.parse(text); } catch (e) { return fallback; }
}

function eventRowToObject_(row) {
  return {
    id: row[0], createdAt: row[1], loggedBy: row[2], timeOfEvent: row[3],
    host: row[4], coHost: row[5], eventType: row[6],
    attendees: safeParseJson_(row[7], []), performance: safeParseJson_(row[8], []),
    whatWentWell: row[9], improvementAreas: row[10], incidentLogId: row[11],
    additionalComments: row[12], reviewedBy: row[13], reviewedAt: row[14]
  };
}

function canSeeEventLog_(user, record) {
  if (user.rankLevel >= 4) return true; // Manager+ see everything
  if (user.rankLevel === 3) {
    const uname = user.username.toLowerCase();
    if ((record.host || '').toLowerCase() === uname) return true;
    if ((record.coHost || '').toLowerCase() === uname) return true;
    return false;
  }
  return false;
}

function submitEventLog(token, data) {
  const user = sessionUser_(token);
  if (!user) return { success: false, message: 'Your session has expired — please sign in again.' };
  if (user.rankLevel < 3) return { success: false, message: 'You do not have permission to log events.' };
  if (!data || !(data.host || '').trim() || !(data.eventType || '').trim()) {
    return { success: false, message: 'Host and event type are required.' };
  }

  const id = Utilities.getUuid();
  getEventLogSheet_().appendRow([
    id, new Date(), user.username,
    data.timeOfEvent || '', data.host.trim(), (data.coHost || '').trim(), data.eventType,
    JSON.stringify(data.attendees || []), JSON.stringify(data.performance || []),
    data.whatWentWell || '', data.improvementAreas || '',
    (data.incidentLogId || '').trim(), data.additionalComments || '',
    '', ''
  ]);
  return { success: true, message: 'Event log submitted.', id: id };
}

function listEventLogs(token) {
  const user = sessionUser_(token);
  if (!user) return { success: false, message: 'Your session has expired — please sign in again.', logs: [] };

  const data = getEventLogSheet_().getDataRange().getValues();
  const logs = [];
  for (let i = 1; i < data.length; i++) {
    const record = eventRowToObject_(data[i]);
    if (canSeeEventLog_(user, record)) logs.push(record);
  }
  logs.sort(function (a, b) { return new Date(b.createdAt) - new Date(a.createdAt); });
  return { success: true, logs: logs };
}

function reviewEventLog(token, id) {
  const user = sessionUser_(token);
  if (!user || user.rankLevel < 4) return { success: false, message: 'Manager+ only.' };

  const sheet = getEventLogSheet_();
  const data = sheet.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === id) {
      sheet.getRange(i + 1, 14).setValue(user.username); // ReviewedBy
      sheet.getRange(i + 1, 15).setValue(new Date());     // ReviewedAt
      return { success: true };
    }
  }
  return { success: false, message: 'Event log not found.' };
}

// =====================================================================
// INCIDENT REPORTS
// =====================================================================
// Visibility: ONLY the Supervisor who logged it, and Manager+. Unlike
// Event Logs, Co-Hosts do NOT get visibility here.
// The "Reviewing Manager" / "Action taken" fields are Manager+-only —
// they don't exist on the initial submission form at all, and are filled
// in (auto-attributed to whichever Manager reviews it) when opened later.

const INCIDENT_LOG_SHEET_NAME = 'IncidentReports';

function getIncidentLogSheet_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(INCIDENT_LOG_SHEET_NAME);
  if (!sheet) {
    sheet = ss.insertSheet(INCIDENT_LOG_SHEET_NAME);
    sheet.appendRow([
      'Id', 'CreatedAt', 'LoggedBy', 'TimeOfEvent', 'Host', 'CoHosts',
      'MembersInvolved', 'IncidentDescription', 'EvidenceLinks',
      'ReviewingManager', 'ActionTakenNotes', 'ReviewedAt'
    ]);
    sheet.setFrozenRows(1);
  }
  return sheet;
}

function incidentRowToObject_(row) {
  return {
    id: row[0], createdAt: row[1], loggedBy: row[2], timeOfEvent: row[3],
    host: row[4], coHosts: row[5], membersInvolved: row[6],
    incidentDescription: row[7], evidenceLinks: row[8],
    reviewingManager: row[9], actionTakenNotes: row[10], reviewedAt: row[11]
  };
}

function submitIncidentReport(token, data) {
  const user = sessionUser_(token);
  if (!user) return { success: false, message: 'Your session has expired — please sign in again.' };
  if (user.rankLevel < 3) return { success: false, message: 'You do not have permission to log incidents.' };
  if (!data || !(data.host || '').trim() || !(data.incidentDescription || '').trim()) {
    return { success: false, message: 'Host and incident description are required.' };
  }

  const id = Utilities.getUuid();
  getIncidentLogSheet_().appendRow([
    id, new Date(), user.username,
    data.timeOfEvent || '', data.host.trim(), (data.coHosts || '').trim(),
    (data.membersInvolved || '').trim(), data.incidentDescription.trim(), (data.evidenceLinks || '').trim(),
    '', '', ''
  ]);
  return { success: true, message: 'Incident report submitted.', id: id };
}

function listIncidentReports(token) {
  const user = sessionUser_(token);
  if (!user) return { success: false, message: 'Your session has expired — please sign in again.', logs: [] };

  const data = getIncidentLogSheet_().getDataRange().getValues();
  const logs = [];
  for (let i = 1; i < data.length; i++) {
    const record = incidentRowToObject_(data[i]);
    const isOwner = record.loggedBy.toLowerCase() === user.username.toLowerCase();
    if (user.rankLevel >= 4 || isOwner) logs.push(record);
  }
  logs.sort(function (a, b) { return new Date(b.createdAt) - new Date(a.createdAt); });
  return { success: true, logs: logs };
}

function reviewIncidentReport(token, id, notes) {
  const user = sessionUser_(token);
  if (!user || user.rankLevel < 4) return { success: false, message: 'Manager+ only.' };

  const sheet = getIncidentLogSheet_();
  const data = sheet.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === id) {
      sheet.getRange(i + 1, 10).setValue(user.username);   // ReviewingManager
      sheet.getRange(i + 1, 11).setValue(notes || '');      // ActionTakenNotes
      sheet.getRange(i + 1, 12).setValue(new Date());       // ReviewedAt
      return { success: true };
    }
  }
  return { success: false, message: 'Incident report not found.' };
}
