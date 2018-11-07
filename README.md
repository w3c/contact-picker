# Contact Picker API
This is a proposal for adding a Contact Picker API to the Web. Contact pickers are frequently seen in native mobile applications for a variety of [use cases](#use-cases), and in various desktop applications such as e-mail clients and calendars.

Contact information is among the most sensitive information on a device, which is why we're proposing a picker rather than a lower level API. This enables user agents to offer an experience that's consistent with platform expectations, and to highlight the exact information that is to be shared with the website.

This proposal does not address the source of the contacts. Many operating systems offer functionality to access the user's contacts, and various browsers allow the user to log in which might provide access to contact information as well.

## Use cases
  * Social networks could use contact information to bootstrap a user's social graph. This is particularly important for emerging markets, where users might prefer a PWA over a native app due to storage constraints.
  * An e-mail application could allow the user to select the recipients for a message without needing their own address book.

## Why is a built-in API better than JavaScript?
  * Excluding the operating system, there is no canonical place where users maintain a list of their contacts. Developers would have to integrate with a variety of services to provide a comprehensive experience.
  * User agents can offer an experience to users that's consistent with other applications on the user's device. The information that will is going to be shared with the website can be clearly highlighted.
  * Contact information is sensitive. Individual contacts can have [any number of properties](https://en.wikipedia.org/wiki/VCard#Properties) associated with them. Websites seeking the e-mail address of a single user should not have access to the same information of all other contacts.

## Example
```javascript
selectRecipientsButton.addEventListener('click', async () => {
  const contacts = await navigator.selectContacts({
      properties: ['name', 'email'],
      multiple: true
  });
    
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
interface ContactInfo {
    readonly attribute string? name;
    readonly attribute sequence<string>? email;
    readonly attribute sequence<string>? tel;
};

enum ContactProperty { "email", "name", "tel" };

dictionary ContactPickerOptions {
    sequence<ContactProperty> properties;
    boolean multiple = false;
};

[Exposed=Window, SecureContext]
partial interface Navigator {
    Promise<sequence<ContactInfo>> selectContacts(ContactPickerOptions options);
};
```

  * Sequences are returned for the properties as multiple may be available in the user's address book. User agents are encouraged to enable users to limit their selection.
  * Support for additional properties can be added iteratively. Whether the returned data can (and should) be sanitized is a question that's unique to each property.

## Security and Privacy
Exposing contact information has a clear privacy impact. We propose a picker model so that the user agent can offer a user experience that makes it clear what information in going to be shared with the website.

The API will only be available on secure contexts. Additionally, we propose that a user gesture is required in order to trigger the contact picker, to help avoid users from inadvertently seeing the picker.

## Potential follow-up work
  * As mentioned, more properties can be added iteratively as use cases are identified. It is a non-goal of this API to share _all_ information stored for a particular contact without filtering.
  * A Contact Provider API can be considered to complement data available from the operating system or the user's account.
