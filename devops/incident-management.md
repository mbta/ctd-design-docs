# Incident Management Guidelines

This document is a supplement to individual runbooks, meant to describe the overall expectations of the on-call user during an incident. Some steps get carried out for all incidents. Others may apply to high-urgency incidents, but not low-urgency ones.

## For All Incidents

### Make sure the incident is in PagerDuty

PagerDuty is the MBTA's source of truth for historical incidents, so every incident must be recorded in PagerDuty for tracking purposes. Usually, incidents are triggered automatically by monitoring systems, but not always.

A new incident can be triggered manually in any of the following ways:

1. By calling our PagerDuty assigned phone number, which will both trigger a new incident and connect you directly to the person currently on call
1. By emailing our PagerDuty assigned email address
1. By clicking the "New Incident" button on the PagerDuty incidents page, if you have a PagerDuty account
1. By typing `/pd trigger` in Slack, if you have a PagerDuty account

### Notify stakeholders

Each runbook has a link to draft an email to all stakeholders of the affected system. Send an email as soon as the incident is confirmed.

### Close the loop

Upon resolving an incident:

1. Send another email to stakeholders confirming that the resolution of the incident
1. Resolve the incident in PagerDuty. Add a brief note with a description of the problem and resolution to the incident. The note should attempt to answer the following questions:
    1. What do we know now?
    1. What do we not know now?
    1. What do we think we know now?
    1. When do we think we will know?

## For High-Urgency Incidents (SEV1/SEV2)

The following steps only apply to incidents that meet the criteria for SEV1 or SEV2 categorization.

### Notify executives

Run the "Notify for Executive Escalation" Response Play to alert the administrative stakeholder on call, who will determine whether to notify executives about the incident.

### Update Slack as you go

Use your #[department]-incident-response Slack channel to share all updates regarding the incident. This will help keep everyone on the same page about what's going on, and possibly prevent duplicate actions from being taken. The log of your communication will get incorporated into the postmortem timeline.

Example updates include:

- **Communication,**, e.g., noting that you sent an email update or contacted a specific person
- **Actions,**, e.g., sharing that you're about to trigger a restart or deploy of an application or perform a specific troubleshooting step
- **Findings,**, e.g., calling out a piece of information that might be relevant to troubleshooting, such as a log message or the state of a system

Encourage other folks who are engaged in incident response to share their updates as well.

### Send hourly email updates

Send updates to stakeholders hourly or as soon as you have any new information to share (whichever comes first). Delegate the hourly updating responsibility as needed.

### Follow-up: Write a postmortem

Every **SEV1** and **SEV2** incident must have a postmortem written within a week of its conclusion. The person who was most closely involved in the incident creates the postmortem but will usually include additional input from anyone else who aided in the response.

You can create a postmortem from an incident by clicking the "+ New Postmortem Report" button on the incident page. The form will walk you through the required information. When the postmortem includes enough information about the incident, create a PDF and post it to the #[department]-incident-response channel in Slack for review by the rest of your team or department.

For more details on PagerDuty's postmortem feature, consult the [PagerDuty Knowledge Base](https://support.pagerduty.com/docs/postmortems).

Once the postmortem document is complete:

- Save it as a PDF and add it to the archive folder
- Create Asana tasks for any pending work raised in the "Prevention" section:
  - Add each task to the Incident After-Actions project for tracking
  - Assign the task to (or tag in) the relevant folks who will triage the task and ultimately complete it
  Request an approximate due date for the task to determine whether to file the request as "Short Term" or "Long Term" section of the Incident After-Actions project.

## For Low-Urgency Incidents (SEV3/SEV4/SEV5)

### Tailor the response to the impact

Low-urgency incidents have a lower expectation for how to respond and how quickly the incident gets resolved. A low-urgency incident is not a "drop everything" situation. However, there are still some actions to take at your earliest convenience. In PagerDuty, an incident categorized as low-urgency will alert using different notification rules and won't escalate if left unacknowledged.

It's still important to verify the issue and notify stakeholders, but any response beyond that can wait until business hours. Product stakeholders may choose to be more proactive in their response, but they are not required to do so.

Individual runbooks should contain more information about the expectations for how to respond to a low-urgency incident.

### Update Slack with the status

Use the #[department]-incident-response Slack channel to share all updates regarding the incident.

### Send email updates as needed

Send email updates to stakeholders as new information is available.

### Follow-up: Track what's needed
