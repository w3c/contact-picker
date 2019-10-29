# Contact Picker API

[Link to spec](https://wicg.github.io/contact-api/spec/)

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

interface ContactAddress : PaymentAddress {};

dictionary ContactInfo {
    sequence<DOMString> name;
    sequence<DOMString> email;
    sequence<DOMString> tel;
    sequence<ContactAddress> address;
    sequence<Blob> icon;
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
Exposing contact information has a clear privacy impact. We propose a picker model so that the user agent can make it clear what information is going to be shared with the website. This differs from native APIs where the permission is requested once, after which the application gets perpetual access to the user's contacts. With the picker model, access to contact information is restricted by the contacts selected by the user in the picker. Furthermore, the spec will have a MUST requirement to ensure that the user can understand which contacts (and information) will be shared.

## Abuse
To prevent abuse and user annoyance, the API will have some usability restrictions. The API will only be available from secure top-level contexts. This will prevent embedded iframes from requesting user contacts and the user accidentally sharing information with an unintended destination. The picker can only be brought up if the API call was initiated by a user gesture. This will prevent websites from bringing up the picker on website load.

## Alternatives Considered

##### `<input type="file" accept="text/contacts+json;items=name,tel" />`
The Web Platform already supports a picker for external data through `<input type="file" />`, which some user agents specialize based on the mime types that are being selected.

Supporting contacts this way has a number of downsides:
  * There is much contention in file types for expressing contact information. For most use cases, a `FileReader` would be used to read and parse the file in order to access the individual properties. We can optimize for this. Libraries can be used for converting this data to a format of the developer's choosing.
  * Feature detection is harder, and would likely need a new API such as `HTMLInputElement.isTypeSupported()` mimicking the [`MediaRecorder`](https://www.w3.org/TR/mediastream-recording/#dom-mediarecorder-istypesupported). Unlike existing specializations of file pickers, contacts would be unlikely to gracefully degrade to a general picker.
  * Extensibility is harder, as it would rely on additional parameters being passed to the mime type.

## Potential follow-up work
  * As mentioned, more properties can be added iteratively as use cases are identified. It is a non-goal of this API to share _all_ information stored for a particular contact without filtering.
  * A Contact Provider API can be considered to complement data available from the operating system or the user's account.
