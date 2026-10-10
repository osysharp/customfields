# Osysharp.CustomFields

Fields your admins add to your records without a release: an order number on a support request, the industry of a
company, the dietary needs of a guest. Each one is text, a number, yes or no, one of a list, or a date. It can be
required, asked of the people outside your team or kept for the team, and grouped on forms whose questions show only
when an earlier answer says so.

Every answer is checked twice. The page checks it as a person types, and the server checks it again with the same rules
before anything is kept. Each answer also holds its field's kind on every write, whoever makes it.

```osy
// app.osy
use Osysharp.CustomFields@0;
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
