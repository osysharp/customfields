# Osysharp.CustomFields

Fields your admins add to your records without a release: an order number on a support request, the industry of a
company, the dietary needs of a guest. Each one is text, a number, yes or no, one of a list, or a date. It can be
required, asked of the people outside your team or kept for the team, and grouped on forms whose questions show only
when an earlier answer says so.

Every answer is checked twice. The page checks it as a person types, and the server checks it again with the same rules
before anything is kept. Each answer also holds its field's kind on every write, whoever makes it.

```osy
// app.osy
use Osysharp.CustomFields@1;
```

## Use it

Say which of your records have fields by giving each kind its own field type and value type. The value carries the
reference to its record, so **a value's security is its record's**: whoever may read a guest may read what you keep
about them, through a rule you write once.

```osy
using Osysharp.CustomFields;

policy ManagesFields => IsAdmin;   // who defines fields and forms

// A guest's fields have names of their own: another kind of record may use the same one.
[Unique(Name, "Another guest field already has that name.")]
entity GuestField : CustomField {
  security { allow read when IsStaff; allow create, update when ManagesFields; }
}

[Unique(Guest, Field, "That guest already has an answer to that field.")]
entity GuestFieldValue : FieldValue {
  [Required] Guest Guest;
  security { allow read, create, update when IsStaff; }
}

// How the editor writes a guest's answers — found without being listed.
public class GuestFields : IFieldRecords {
  public bool Holds(CustomField field) => field is GuestField;
  public FieldValue? AnswerOf(Guid record, CustomField field) =>
    GuestFieldValue.FirstOrDefault(v => v.Guest.Id == record && v.Field == field);
  public FieldValue NewAnswer(Guid record, CustomField field) =>
    new GuestFieldValue { Guest = Guest.Single(g => g.Id == record), Field = field };
  public string LastChangedBy(Guid record) => "Someone";
  public void Changed(Guid record, CustomField field, string before, string after) { }
}
```

Show a guest's fields, each edited in place:

```osy
[Route("/guests/{id}")] [Authorize(IsStaff)]
component GuestPage(Guid id) {
  live var fields = GuestField.OrderBy(f => f.Position)
    .Select(f => new FieldSpec { Id = f.Id, Name = f.Name, Kind = f.Kind, Options = f.Options, Hint = f.Hint,
                                 Public = f.Public, Required = f.Required, Active = f.Active }).ToList();
  live var held = GuestFieldValue.Where(v => v.Guest.Id == id)
    .Select(v => new FieldAnswer { Field = v.Field.Id, Value = v.Value }).ToList();
  render { RecordFields(id, fields, held, title: "About this guest"); }
}
```

And let your admins define them, on a settings page of your own:

```osy
[Route("/settings/fields")] [Authorize(ManagesFields)]
component FieldsSettings() {
  live var fields = GuestField.OrderBy(f => f.Position).ToList();
  live var used = GuestFieldValue.Where(v => v.Value != "").GroupBy(v => v.Field.Id)
    .Select(g => new FieldUse { Field = g.Key, Count = g.Count() }).ToList();
  CustomField AddGuestField(FieldDraft d) => FillField(new GuestField(), d, fields.Count);
  render {
    SettingsPage("Fields") {
      FieldListSection(fields, AddGuestField, used, title: "Guest fields", record: "guest", records: "guests");
      SaveBar(saved: "Fields saved.");
    }
  }
}
```

## Forms

A form is a kind of request, a type of booking, a kind of application. It asks some of the fields, in its own order, and
each question can be shown on one condition: "show it when *Did you see an error?* is *yes*". Derive your own form type,
or use `FieldForm` as it is. `FormSection` edits one form on your settings page, and anything you add inside its body
(`{ … }`) appears under its key. `Checked(form, answers, outside: true)` is what your server function runs on an
outside person's answers. It refuses an answer to anything the form does not ask them, drops what a condition hides,
and names every required question left empty.

## What it does

| | |
|---|---|
| **Five kinds** | text, a number, yes or no, one of a list (short lists as buttons, long ones as a menu), a date. One spelling per kind: "yes", "42", "2026-10-02", the option as written |
| **Retired, never deleted** | a retired field stops being asked and edited, and every record keeps its answer, shown as retired. A field's kind is fixed once saved |
| **Asked or the team's** | a public field is asked on a form; the rest are the team's, edited on the record |
| **Conditions** | a question shows when an earlier one has a given answer; the settings page says when a condition can never be met, and why |
| **No silent overwrite** | an edit carries the answer the page showed. If a colleague changed it since, the save is refused, and the message names who and what it is now |
| **Checked on the server** | the page and the server ask the same pure functions; the value's own invariants hold its kind on every write |

## In reports

On an app that also has a report builder (`Osysharp.Reports`), every field is a column of its record: "Guest ›
Dietary needs" groups, filters and is measured like any field the app declares, as its kind reads. A list groups by its
options, yes or no reads Yes and No, a number sums and compares, and a date groups by week or month. Declare the
relation back from the record to its answers, beside the one-answer-per-field rule:

```osy
entity Guest {
  [ForeignKey(Guest)] GuestFieldValue[] Fields;
}
```

A reader is offered a field only when they may read an answer to it.

## Personal data

An answer can be about a person, so the kit classifies the value `Personal.Content`. Your value type says which category
it is held under and whose it is. An erasure blanks the answer, like any personal field the row holds; to remove the
whole row instead, say so:

```osy
[PersonalData(GuestDetails, OnErasure.DeleteRow)]
entity GuestFieldValue : FieldValue { [Required, DataSubject] Guest Guest; … }
```

## What an app calls

| | |
|---|---|
| `SetField(record, field, value, expected)` | answer, change or clear one field, checked, written through your `IFieldRecords` |
| `Checked(form, answers, outside)` · `CheckedChange(field, before, value, expected)` | the server's checks |
| `ShownFields(onForm, answers, outside)` · `AnswerProblem(spec, value)` · `Normalised(spec, value)` · `ShownAnswer(kind, value)` | the rules, both sides |
| `FormSpecs()` · `SpecOf(field)` · `FillField(field, draft, position)` · `FieldNameProblem(name, self, siblings)` | |
| `Question` · `Choices` · `FieldRow` · `RecordFields` · `FieldAnswerLines` | the controls and the editor |
| `FieldListSection` · `FormSection` · `ConditionEditor` | the settings sections |

## Its tests

The kit's test app is a photography studio's bookings. A wedding asks for the venue, the number of guests and the date;
a portrait asks whether it is for a business, and the company's name only when it is. The team keeps its own notes on a
booking, and the studio keeps a client's preferred contact time, which the client never sees. Every rule is checked on
the server and on a direct write, and the settings page and the editor are driven as a person would use them, wide and
on a phone.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.CustomFields;
using FieldsKitTests;

[Test(Studio)]
void a_question_on_a_condition_shows_only_once_the_earlier_answer_says_so() {
  runas (Cleo) {
    var portrait = FormSpecs().Where(x => x.Form == KindId("portrait")).ToList();
    Assert.Equal(1, ShownFields(portrait, new List<FieldAnswer>(), true).Count);
    Assert.Equal(1, ShownFields(portrait, [Said("For a business?", "no")], true).Count);
    Assert.Equal(2, ShownFields(portrait, [Said("For a business?", "Yes")], true).Count);
  }
}
```

From [`tests/fields.test.osy`](tests/fields.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.CustomFields;
using FieldsKitTests;

[Test(Studio)]
void every_kind_says_what_is_wrong_in_words_and_keeps_one_spelling() {
  var guests = new FieldSpec { Name = "Guests", Kind = FieldKind.Number };
  Assert.Equal("Guests takes a number, like 42 or 3.5.", AnswerProblem(guests, "forty"));
  Assert.Equal("", AnswerProblem(guests, " 120 "));
  Assert.Equal("120", Normalised(guests, " 120 "));
  var on = new FieldSpec { Name = "Shoot on", Kind = FieldKind.Date };
  Assert.Equal("Shoot on takes a date, like 2026-10-02.", AnswerProblem(on, "2026-02-30"));
  Assert.Equal("2028-02-29", Normalised(on, "2028-02-29"));
  Assert.Equal("29 Feb 2028", ShownAnswer(FieldKind.Date, "2028-02-29"));
  var lens = new FieldSpec { Name = "Lens", Kind = FieldKind.Choice, Options = "35mm\n50mm\n85mm" };
  Assert.Equal("Lens is one of: 35mm, 50mm, 85mm.", AnswerProblem(lens, "24mm"));
  Assert.Equal("85mm", Normalised(lens, "85MM"));
  var yes = new FieldSpec { Name = "Deposit paid", Kind = FieldKind.YesNo };
  Assert.Equal("Deposit paid is yes or no.", AnswerProblem(yes, "maybe"));
  Assert.Equal("yes", Normalised(yes, "Yes"));
  Assert.Equal("order-number", FieldWord(" Order Number "));
  // A field nobody answered says nothing about answers — an empty FirstOrDefault of a count is 0, not null.
  Assert.Equal("", UseLine(new List<FieldUse>(), Guid.NewGuid(), "booking", "bookings"));
  Assert.Equal(" · answered on 2 bookings", UseLine([new FieldUse { Field = Guid.Empty, Count = 2 }], Guid.Empty, "booking", "bookings"));
}
```

From [`tests/fields.test.osy`](tests/fields.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.CustomFields;
using FieldsKitTests;

[Test(Studio)]
void a_client_books_and_the_server_keeps_only_what_the_form_asks_them() {
  runas (Cleo) {
    var cleo = ClientId("Cleo Martin");
    // A required question left empty is refused, naming it — and every problem is said at once.
    var refused = Assert.Throws<ValidationException>(() => BookShoot(cleo, "Our wedding", KindId("wedding"), [Said("Guests", "lots")]));
    Assert.Contains("Venue needs an answer.", refused.Message);
    Assert.Contains("Guests takes a number", refused.Message);
    // A team-only field is not the client's to answer, whatever the page sent.
    Assert.Throws<ValidationException>(() => BookShoot(cleo, "Our wedding", KindId("wedding"), [Said("Venue", "The old mill"), Said("Deposit paid", "yes")]));
    // A field from another form is refused too.
    Assert.Throws<ValidationException>(() => BookShoot(cleo, "Our wedding", KindId("wedding"), [Said("Venue", "The old mill"), Said("For a business?", "no")]));
    var id = BookShoot(cleo, "Our wedding", KindId("wedding"), [Said("Venue", " The old mill "), Said("Guests", "80"), Said("Shoot on", "2027-06-12")]);
    Assert.Equal(3, BookingFieldValue.Count(v => v.Booking.Id == id));
    Assert.Equal("The old mill", BookingFieldValue.Single(v => v.Booking.Id == id && v.Field.Name == "Venue").Value);
  }
  // A condition's hidden answer is dropped, not kept.
  runas (Cleo) {
    var id = BookShoot(ClientId("Cleo Martin"), "Headshots", KindId("portrait"), [Said("For a business?", "no"), Said("Company name", "Martin & Co")]);
    Assert.Equal(1, BookingFieldValue.Count(v => v.Booking.Id == id));
  }
}
```

From [`tests/fields.test.osy`](tests/fields.test.osy).

## Learn more

- [The Osysharp.CustomFields reference](https://osysharp.com/reference/ui/fields-kit/)
- [Osysharp.CustomFields in the package catalogue](https://osysharp.com/packages/#osysharp-customfields)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
