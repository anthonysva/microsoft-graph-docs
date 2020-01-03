---
title: "Share or delegate a calendar in Outlook"
description: "In Outlook, a calendar owner can share the calendar with another user, or delegate another user to manage meetings in the owner's primary calendar."
author: "angelgolfer-ms"
localization_priority: Priority
ms.prod: "outlook"
---

# Share or delegate a calendar in Outlook (preview)

In Outlook, a calendar owner can share the calendar with another user, or delegate another user to manage meetings in the owner's primary calendar. The owner can specify which information on non-private events is viewable, and can give write access to the calendar to users in the same organization. 

Depending on the permission assigned by the owner, sharees can view permitted information in, or edit non-private events. Delegates are sharees who can view all information in and have write access to non-private events; they also receive meeting requests and responses, and respond to meeting requests on behalf of the owner. Additionally, the owner can give explicit permissions to delegates to view the owner's _private_ events on the calendar. 

Before calendar sharing or delegation takes effect, the owner sends a sharee or delegate an invitation, and the sharee or delegate has to accept the invitation. All this occurs in an Outlook client. After the sharing or delegation takes effect, apps can use the Microsoft Graph API to do the following tasks:

- [Get information about the sharees and delegates, and the allowed permissions, and get or set the actual permissions to a calendar](#get-and-set-information-about-sharees-delegates-and-permissions-for-a-calendar).
- [Get the properties that describe the sharing or delegation of the calendar](#get-properties-of-a-shared-or-delegated-calendar).
- [Get or set the mailbox setting for the calendar owner to continue to receive meeting requests and responses for a delegated calendar](#get-or-set-the-mailbox-setting-for-calendar-owner-to-receive-meeting-requests-and-responses).
- [Delete a sharee or delegate of a calendar](#delete-a-sharee-or-delegate-of-a-calendar).

Note that most of the API for sharing or delegation is in preview and available only in the beta version.

In the following series of examples:

- Alex Wilber has delegated Megan Bowen to his primary calendar, and also permitted Megan to view his private calendar items. 
- Alex shared a "Kids parties" calendar with Adele Vance and Megan Bowen, and gave both of them `read` permissions to all the details of the events on the "Kids parties" calendar.


## Get and set information about sharees, delegates, and permissions for a calendar

In this section:

- [Get permissions and sharing or delegation information on behalf of owner](#get-permissions-and-sharing-or-delegation-information-on-behalf-of-owner)
- [Set permissions for an existing sharee or delegate on a calendar](#set-permissions-for-an-existing-sharee-or-delegate-on-a-calendar)

Each calendar is associated with a collection of [calendarPermission](/graph/api/resources/calendarpermission?view=graph-rest-beta) objects, each of which describes any sharee or delegate and the associated permission that the calendar owner has set up. The [calendarRoleType](/graph/api/resources/calendarpermission#calendarroletype-values?view=graph-rest-beta) enumeration defines the range of permissions that Microsoft Graph supports:

- `none`
    The calendar is not shared with or delegated to anyone.
- `freeBusyRead`
    The sharee can view the owner's free/busy status on the calendar.
- `limitedRead`
    The sharee can view the owner's free/busy status, and the titles and locations of the events on the calendar.
- `read`
    The sharee can view all the details of the events on the calendar.
- `write`
    The sharee can view all the details and edit (create, update, or delete) events on the calendar.
- `delegateWithoutPrivateEventAccess`
    The delegate has `write` access but cannot view information of the owner's private events on the calendar.
- `delegateWithPrivateEventAccess`
    The delegate has `write` access and can view information of the owner's private events on the calendar.

The primary calendar of a user is always shared with "My Organization", which represents the users in the same organization as the owner. By default, they can read the owner's free/busy status on that calendar and have the `freeBusyRead` permission.


### Get permissions and sharing or delegation information on behalf of owner

The following example gets the **calendarPermission** objects associated with Alex' primary calendar. The request returns two such permission objects:

- In the first **calendarPermission** object assigned to the delegate, Megan:

  - **isRemovable** is set to true, providing Alex the option to cancel the delegation 
  - **isInsideOrganization** is true as only users in the same organization can be delegates
  - **role** for Megan is `delegateWithPrivateEventAccess`, as set up by Alex
  - **allowedRoles** includes the role types `delegateWithoutPrivateEventAccess` and `delegateWithPrivateEventAccess` that support delegation
  - **emailAddress** specifies Megan

- In the second **calendarPermission** object assigned to "My Organization":

  - **isRemovable** is set to false, since the primary calendar is always shared with the owner's organization 
  - **isInsideOrganization** is true
  - **role** is `freeBusyRead`, as Outlook sets up by default for "My Organization"
  - **emailAddress** specifies the **name** sub-property as "My Organization"; **address** for "My Organization" is by default null.

<!-- {
  "blockType": "request",
  "name": "get_calendarperms"
}-->
```http
GET https://graph.microsoft.com/beta/users/AlexW@contoso.OnMicrosoft.com/calendar/calendarPermissions
```

<!-- {
  "blockType": "response",
  "name": "get_calendarperms",
  "truncated": true,
  "@odata.type": "microsoft.graph.calendarPermission",
  "isCollection": true
} -->
```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users('64339082-ed84-4b0b-b4ab-004ae54f3747')/calendar/calendarPermissions",
    "value": [
        {
            "id": "L289RXhjaGFuZ2VMYWJTWVnYW5C",
            "isRemovable": true,
            "isInsideOrganization": true,
            "role": "delegateWithPrivateEventAccess",
            "allowedRoles": [
                "freeBusyRead",
                "limitedRead",
                "read",
                "write",
                "delegateWithoutPrivateEventAccess",
                "delegateWithPrivateEventAccess"
            ],
            "emailAddress": {
                "name": "Megan Bowen",
                "address": "MeganB@contoso.OnMicrosoft.com"
            }
        },
        {
            "id": "RGVmYXVsdA==",
            "isRemovable": false,
            "isInsideOrganization": true,
            "role": "freeBusyRead",
            "allowedRoles": [
                "none",
                "freeBusyRead",
                "limitedRead",
                "read",
                "write"
            ],
            "emailAddress": {
                "name": "My Organization"
            }
        }
    ]
}
```


### Set permissions for an existing sharee or delegate on a calendar

You can update the permissions assigned to an existing sharee or delegate, as long as the new permissions are supported by those **allowedRoles** set up initially for the sharee or delegate for that calendar. 

You cannot update any other sharee or delegate information once the sharing or delegation has been set up, including **isRemovable**, **isInsideOrganization**, **allowedRoles**, and **emailAddress**. Changing any of these properties requires deleting the sharee or delegate and setting up a new instance of **calendarPermission** again.

The following example changes the permission of an existing sharee, Adele, from `read` to `write`, on behalf of the calendar owner, Alex, for the custom calendar "Kids parties".

<!-- {
  "blockType": "request",
  "name": "update_calendarperm",
  "sampleKeys": ["AlexW@contoso.OnMicrosoft.com", "AAMkADAwAABf02bAAAA=", "L289RXhjaGFuZ2VMYWJQWRlbGVW"]
}-->
```http
PATCH https://graph.microsoft.com/beta/users/AlexW@contoso.OnMicrosoft.com/calendars/AAMkADAwAABf02bAAAA=/calendarPermissions/L289RXhjaGFuZ2VMYWJQWRlbGVW
Content-type: application/json

{
  "role": "write"
}
```

<!-- {
  "blockType": "response",
  "name": "update_calendarperm",
  "truncated": true,
  "@odata.type": "microsoft.graph.calendarPermission"
} -->

```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users('64339082-ed84-4b0b-b4ab-004ae54f3747')/calendars('AAMkADAwAABf02bAAAA%3D')/calendarPermissions/$entity",
    "id": "L289RXhjaGFuZ2VMYWJQWRlbGVW",
    "isRemovable": true,
    "isInsideOrganization": true,
    "role": "write",
    "allowedRoles": [
        "freeBusyRead",
        "limitedRead",
        "read",
        "write"
    ],
    "emailAddress": {
        "name": "Adele Vance",
        "address": "AdeleV@contoso.OnMicrosoft.com"
    }
}
```


## Get properties of a shared or delegated calendar

In this section:

- [Get properties of a shared or delegated calendar on behalf of owner](#get-properties-of-a-shared-or-delegated-calendar-on-behalf-of-owner)
- [Get properties of shared or delegated calendar on behalf of sharee or delegate](#get-properties-of-shared-or-delegated-calendar-on-behalf-of-sharee-or-delegate)

Recalling in this example, Alex has delegated his primary calendar and given the delegate, Megan Bowen, the permission to view calendar items that are marked private.
This section shows the calendar properties for the owner, Alex, and for the delegate, Megan.

### Get properties of a shared or delegated calendar on behalf of owner

The following example gets the properties of the primary calendar on behalf of the owner, Alex. 

Note the following properties on Alex' behalf:

- **canShare** is true as Alex is the owner.
- **canViewPrivateItems** is true since Alex is the owner.
- **isShared** is set to true, as Alex has set up a delegate for this calendar.
- **isSharedWithMe** is always false for the calendar owner.
- **owner** shows Alex as the owner.


<!-- {
  "blockType": "request",
  "name": "get_calendar_props_owner",
  "sampleKeys": ["AlexW@contoso.OnMicrosoft.com"]
}-->
```http
GET https://graph.microsoft.com/beta/users/AlexW@contoso.OnMicrosoft.com/calendar
```

<!-- {
  "blockType": "response",
  "name": "get_calendar_props_owner",
  "truncated": true,
  "@odata.type": "microsoft.graph.calendar"
} -->
```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users('64339082-ed84-4b0b-b4ab-004ae54f3747')/calendar/$entity",
    "id": "AQMkADAw7QAAAJfygAAAA==",
    "name": "Calendar",
    "color": "auto",
    "hexColor": "",
    "isDefaultCalendar": true,
    "changeKey": "NEXywgsVrkeNsFsyVyRrtAAAAAACOg==",
    "canShare": true,
    "canViewPrivateItems": true,
    "isShared": true,
    "isSharedWithMe": false,
    "canEdit": true,
    "allowedOnlineMeetingProviders": [
        "teamsForBusiness"
    ],
    "defaultOnlineMeetingProvider": "teamsForBusiness",
    "isTallyingResponses": true,
    "isRemovable": false,
    "owner": {
        "name": "Alex Wilber",
        "address": "AlexW@contoso.OnMicrosoft.com"
    }
}
```


### Get properties of shared or delegated calendar on behalf of sharee or delegate

The following example gets the properties of the same calendar on behalf of the delegate, Megan. 

Note the following properties:

- **name** of the calendar is "Alex Wilber", since this is Alex' calendar delegated to Megan.
- **canShare** is false, since Megan is not the owner of this calendar.
- **canViewPrivateItems** is true for the delegate Megan, as set up by Alex. For a sharee that is not a delegate, this property is always false.
- **isShared** is false. This property indicates only to a calendar _owner_ whether the calendar has been shared or delegated.
- **isSharedWithMe** property is true, since Megan is a delegate.
- **canEdit** is true, since delegates, including Megan, have write access.
- **owner** is set to Alex.

<!-- {
  "blockType": "request",
  "name": "get_calendar_props_delegate",
  "sampleKeys": ["meganb@contoso.OnMicrosoft.com", "AAMkADlAABhbftjAAA="]
}-->
```http
GET https://graph.microsoft.com/beta/users/meganb@contoso.OnMicrosoft.com/calendars/AAMkADlAABhbftjAAA=
```

<!-- {
  "blockType": "response",
  "name": "get_calendar_props_delegate",
  "truncated": true,
  "@odata.type": "microsoft.graph.calendar"
} -->
```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users('meganb%40contoso.OnMicrosoft.com')/calendars/$entity",
    "id": "AAMkADlAABhbftjAAA=",
    "name": "Alex Wilber",
    "color": "auto",
    "hexColor": "",
    "isDefaultCalendar": false,
    "changeKey": "E6LznKWmX0KTsAD9qRJjeAAAYWo3EQ==",
    "canShare": false,
    "canViewPrivateItems": true,
    "isShared": false,
    "isSharedWithMe": true,
    "canEdit": true,
    "allowedOnlineMeetingProviders": [
        "teamsForBusiness"
    ],
    "defaultOnlineMeetingProvider": "teamsForBusiness",
    "isTallyingResponses": true,
    "isRemovable": true,
    "owner": {
        "name": "Alex Wilber",
        "address": "AlexW@contoso.OnMicrosoft.com"
    }
}
```


## Get or set the mailbox setting for calendar owner to receive meeting requests and responses

In this section:

- [Get delegation delivery setting for a user's mailbox](#get-delegation-delivery-setting-for-a-users-mailbox)
- [Set delegation delivery setting for a user's mailbox](#set-delegation-delivery-setting-for-a-users-mailbox)

Depending on the level of delegation a calendar owner prefers, the owner can specify who should receive meeting requests and responses to manage meetings on the calendar. 

Programmatically, you can get or set the **delegateMeetingMessageDeliveryOptions** property of the calendar owner's [mailboxSettings](/graph/api/resources/mailboxsettings?view=graph-rest-beta) to specify to whom Outlook should direct [eventMessageRequest](/graph/api/resources/eventmessagerequest?view=graph-rest-beta) and [eventMessageResponse](/graph/api/resources/eventmessageresponse?view=graph-rest-beta) instances:

- `sendToDelegateOnly`

    Outlook to direct **eventMessageRequest** and **eventMessageResponse** instances to only delegates. This is the default setting.
- `sendToDelegateAndInformationToPrincipal`

    Outlook to direct **eventMessageRequest** and **eventMessageResponse** instances to delegates and the calendar owner. Only the delegates see the option to accept or decline a meeting request, and the notification sent to the owner appears like a normal email message. The owner can still respond to the meeting by opening the calendar item and responding.
- `sendToDelegateAndPrincipal`

    Outlook to direct **eventMessageRequest** and **eventMessageResponse** instances to delegates and the calendar owner, either of whom can respond to the meeting request.

The same setting applies to all the delegates the owner has set up for the primary calendar.

### Get delegation delivery setting for a user's mailbox

The following example gets the **mailboxSettings** of a calendar owner who lets Outlook direct meeting requests and responses to only calendar delegates; that is, **delegateMeetingMessageDeliveryOptions** is set to `sendToDelegateOnly`.

<!-- {
  "blockType": "request",
  "name": "get_mailboxsettings_owner",
  "sampleKeys": ["AlexW@contoso.OnMicrosoft.com"]
}-->
```http
GET https://graph.microsoft.com/beta/users/AlexW@contoso.OnMicrosoft.com/mailboxsettings
```

<!-- {
  "blockType": "response",
  "name": "get_mailboxsettings_owner",
  "truncated": true,
  "@odata.type": "microsoft.graph.mailboxSettings"
} -->
```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users('64339082-ed84-4b0b-b4ab-004ae54f3747')/mailboxSettings",
    "archiveFolder": "AQMkADAwAGVQAAAKfowAAAA==",
    "timeZone": "Pacific Standard Time",
    "delegateMeetingMessageDeliveryOptions": "sendToDelegateOnly",
    "dateFormat": "M/d/yyyy",
    "timeFormat": "h:mm tt",
    "automaticRepliesSetting": {
        "status": "disabled",
        "externalAudience": "all",
        "internalReplyMessage": "",
        "externalReplyMessage": "",
        "scheduledStartDateTime": {
            "dateTime": "2019-12-24T05:00:00.0000000",
            "timeZone": "UTC"
        },
        "scheduledEndDateTime": {
            "dateTime": "2019-12-25T05:00:00.0000000",
            "timeZone": "UTC"
        }
    },
    "language": {
        "locale": "en-US",
        "displayName": "English (United States)"
    },
    "workingHours": {
        "daysOfWeek": [
            "monday",
            "tuesday",
            "wednesday",
            "thursday",
            "friday"
        ],
        "startTime": "08:00:00.0000000",
        "endTime": "17:00:00.0000000",
        "timeZone": {
            "name": "Pacific Standard Time"
        }
    }
}
```

### Set delegation delivery setting for a user's mailbox

The following example updates the **delegateMeetingMessageDeliveryOptions** property to `sendToDelegateAndPrincipal`, to have Outlook direct meeting requests and responses of the delegated calendar to all delegates and the owner.

!-- {
  "blockType": "request",
  "name": "patch_mailboxsettings_owner",
  "sampleKeys": ["AlexW@contoso.OnMicrosoft.com"]
}-->
```http
PATCH https://graph.microsoft.com/beta/users/AlexW@contoso.OnMicrosoft.com/mailboxsettings
Content-type: application/json

{
  "delegateMeetingMessageDeliveryOptions": "sendToDelegateAndPrincipal"
}
```

<!-- {
  "blockType": "response",
  "name": "patch_mailboxsettings_owner",
  "truncated": true,
  "@odata.type": "microsoft.graph.mailboxSettings"
} -->
```http
HTTP/1.1 200 OK
Content-type: application/json

{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users('64339082-ed84-4b0b-b4ab-004ae54f3747')/mailboxSettings",
    "delegateMeetingMessageDeliveryOptions": "sendToDelegateAndPrincipal"
}
```


## Delete a sharee or delegate of a calendar

In the following example, Alex deletes Megan as a sharee of the "Kids parties" calendar.

!-- {
  "blockType": "request",
  "name": "delete_sharee",
  "sampleKeys": ["AlexW@contoso.OnMicrosoft.com", "AAMkADAwAABf02bAAAA=", "L289RXhjaGFuZ2VMYWJTWVnYW5C"]
}-->
```http
DELETE https://graph.microsoft.com/beta/users/AlexW@contoso.OnMicrosoft.com/calendars/AAMkADAwAABf02bAAAA=/calendarPermissions/L289RXhjaGFuZ2VMYWJTWVnYW5C
```

<!-- {
  "blockType": "response",
  "name": "delete_sharee",
  "truncated": true
} -->
```http
HTTP/1.1 204 No Content
```


## Next steps

Find out more about:

- How the Outlook clients support sharing and delegating calendars:
  - [Share an Outlook calendar with other people](https://support.office.com/article/share-an-outlook-calendar-with-other-people-353ed2c1-3ec5-449d-8c73-6931a0adab88
)
  - [Allow someone else to manage your mail and calendar as a delegate](https://support.office.com/article/allow-someone-else-to-manage-your-mail-and-calendar-41c40c04-3bd1-4d22-963a-28eafec25926)
  - [Share your calendar in Outlook on the web](https://support.office.com/article/share-your-calendar-in-outlook-on-the-web-7ecef8ae-139c-40d9-bae2-a23977ee58d5)
  - [Calendar delegation in Outlook on the web](https://support.office.com/article/calendar-delegation-in-outlook-on-the-web-532e6410-ee80-42b5-9b1b-a09345ccef1b
)
- [Why integrate with Outlook calendar](outlook-calendar-concept-overview.md)
- The [calendar API](/graph/api/resources/calendar?view=graph-rest-1.0) in Microsoft Graph v1.0.