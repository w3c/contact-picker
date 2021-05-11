# Contact Picker API

[Link to spec](https://wicg.github.io/contact-api/spec/index.html)

This is a proposal for adding a Contact Picker API to the Web. Contact pickers are frequently seen in native mobile applications for a variety of [use cases](#use-cases), and in various desktop applications such as e-mail clients and calendars.

Contact information is among the most sensitive information on a device, which is why we're proposing a picker rather than a lower level API. This enables user agents to offer an experience that's consistent with platform expectations, and to highlight the exact information that is to be shared with the website.

This proposal does not address the source of the contacts. Many operating systems offer functionality to access the user's contacts, and various browsers allow the user to log in which might provide access to contact information as well.

## Use cases
  * Social networks could use contact information to bootstrap a user's social graph. This is particularly important for emerging markets, where users might prefer a PWA over a native app due to storage constraints.
  * An e-mail application could allow the user to select recipients for a message without needing its own address book.

## Why do we need a new built-in API?
  * Excluding the operating system, there is no canonical place where users maintain a list of their contacts. Developers would have to integrate with a variety of services to provide a comprehensive experience.
  * User agents can offer an experience to users that's consistent with other applications on the user's device. The information that will be shared with the website can be clearly highlighted.
  * Contact information is sensitive. Individual contacts can have [any number of properties](https://en.wikipedia.org/wiki/VCard#Properties) associated with them. Websites seeking the e-mail address of a single user should not have access to all other contacts.

## Example
```javascript
selectRecipientsButton.addEventListener('click', async () => {
  const contacts = await navigator.contacts.select(['name', 'email'], { multiple: true });
    
  if (!contacts.length) {
    // Either no contacts were selected in the picker, or the picker could
    // not be launched. Exposure of the API implies expected availability.
    return;
  }
  
  // Use the names and e-mail addresses in |contacts| to populate the
  // recipients field in the website's UI.
  populateRecipients(contacts);
});
```

## Proposed WebIDL
```WebIDL
enum ContactProperty { "address", "email", "icon", "name", "tel" };

interface ContactAddress {
  [Default] object toJSON();
  readonly attribute DOMString city;
  readonly attribute DOMString country;
  readonly attribute DOMString dependentLocality;
  readonly attribute DOMString organization;
  readonly attribute DOMString phone;
  readonly attribute DOMString postalCode;
  readonly attribute DOMString recipient;
  readonly attribute DOMString region;
  readonly attribute DOMString sortingCode;
  readonly attribute FrozenArray<DOMString> addressLine;
};

dictionary ContactInfo {
    sequence<ContactAddress> address;
    sequence<DOMString> email;
    sequence<Blob> icon;
    sequence<DOMString> name;
    sequence<DOMString> tel;
};

dictionary ContactsSelectOptions {
    boolean multiple = false;
};

[Exposed=(Window,SecureContext)]
interface ContactsManager {
    Promise<sequence<ContactProperty>> getProperties();
    Promise<sequence<ContactInfo>> select(sequence<ContactProperty> properties, optional ContactsSelectOptions options);
};

[Exposed=Window]
partial interface Navigator {
  [SecureContext, SameObject] readonly attribute ContactsManager contacts;
};
```

  * Sequences are returned for the properties as multiple values may be available in the user's address book. User agents are encouraged to enable users to limit their selection.
  * Support for additional properties can be added iteratively. Whether the returned data can (and should) be sanitized is a question that's unique to each property. The user agent must provide at least one property.
  * Some future might include the ability to add contacts, or even _contact management_, so having an intermediary object on `navigator` helps extensibility.

## Security and Privacy
Exposing contact information has a clear privacy impact. The spec proposes a picker model so that the user agent can make it clear what information is going to be shared with the website. This differs from native APIs where the permission is requested once, after which the application gets perpetual access to the user's contacts. With the picker model, the website gets a one-off list of contacts selected by the user. Furthermore, The API will only be available from secure top-level contexts.

## Abuse Scenarios

### Sharing contacts with unintended recipients
The first mitigation against this abuse vector is that the API is only available from top-level contexts. This means embedded iframes can't request contact information.

The spec also enforces that the picker UI shows which origin the contacts will be shared with, which contacts will be shared, and what information regarding the contacts will be shared. 

### Websites forcing users to share contacts
The picker UI MUST have a cancel/return option for the user to not share any contacts. The spec also recommends that the UI have a way for users to opt out of sharing certain contact information, in a way that the website cannot know whether the user opted out.

Implementers should be wary of malicious website UI that can be rendered side-by-side with the picker. The UI can pretend to be part of the user agent's UI and encourage users to share more information than they intended to. A potential workaround to avoid this would be making the picker fullscreen.

### Websites spamming users with the picker
To prevent abuse and user annoyance, the API has some usability restrictions. The picker can only be brought up if the API call was initiated by a user gesture. This means websites can't bring up the picker on website load. There can also be one instance of the picker at any given time, so websites can't request multiple stacked pickers on top of each other. 

## Alternatives Considered

##### `<input type="file" accept="text/contacts+json;items=name,tel" />`
The Web Platform already supports a picker for external data through `<input type="file" />`, which some user agents specialize based on the mime types that are being selected.

Supporting contacts this way has a number of downsides:
  * There is much contention in file types for expressing contact information. For most use cases, a `FileReader` would be used to read and parse the file in order to access the individual properties. We can optimize for this. Libraries can be used for converting this data to a format of the developer's choosing.
  * Feature detection is harder, and would likely need a new API such as `HTMLInputElement.isTypeSupported()` mimicking the [`MediaRecorder`](https://www.w3.org/TR/mediastream-recording/#dom-mediarecorder-istypesupported). Unlike existing specializations of file pickers, contacts would be unlikely to gracefully degrade to a general picker.
  * Extensibility is harder, as it would rely on additional parameters being passed to the mime type.

#### Previous Standardization Attempts
There have been multiple standardization attempts to bring a Contacts API to the web. Here's a list of them and brief explanation of why they never materialized.
  * https://lists.w3.org/Archives/Public/public-device-apis/2009Apr/att-0001/contacts.html
      
      This attempt was _donated_ by Nokia rather than being formally put on the standardization track, and it seems to have not been picked up since.

  * https://www.w3.org/TR/2010/WD-contacts-api-20100121/
      
      This attempt was shelved in 2014 in favour of Web Intents, which never materialized.

  * https://www.w3.org/TR/2015/NOTE-contacts-manager-api-20150602/
      
      This attempt built on top of the 2010 one by adding more programmatic access to the user's address book. This seems to be quite distantly separated from today's interest from implementations.

And some proprietary attempts.
  * https://wiki.mozilla.org/WebAPI/ContactsAPI
      
      This attempt was not put on the standardization track, and doesn't seem to have been picked up since.

The current proposal differs in its approach to privacy, which was the main emphasis of the API when it was designed. Unlike previous attempts which allow for perpetual access after granted permission, or include a vague privacy model, this spec enforces UI restrictions which give users full control over shared data and limit abuse. For example, a picker model is enforced where the user always acts as an intermediary of the shared contact info with full control every time contacts are requested.

## Potential follow-up work
  * As mentioned, more properties can be added iteratively as use cases are identified. It is a non-goal of this API to share _all_ information stored for a particular contact without filtering.
  * A Contact Provider API can be considered to complement data available from the operating system or the user's account.
