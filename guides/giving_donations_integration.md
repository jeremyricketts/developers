# Giving Donations Integration

Because of the way that the Planning Center apps are structured in relation to each other and how Giving itself is built, there are a few concepts that are particularly helpful to know when building any sort of API integration that creates Donation records in Giving, especially on an ongoing basis.

## Donation Essentials

### `Donation`

Most of the information you'll need about `Donation`s can be found in the [API docs](https://developer.planning.center/docs/#/apps/giving/2018-08-01/vertices/donation), and the general concept should be pretty clear.  This is the object that corresponds to a gift given to an organization at a particular point in time.  It ties the money given to the person who gave the money, how the money should be designated to one or more funds, and what means they used to give the gift (e.g. system originating the transaction, method of funding the transaction, etc.).

### `Fund`

Each organization will have a set of one or more `Fund`s in their account that serve as buckets that donor's can effectively put money into and that the churches can use to organize their incoming gifts, set budgets, etc. All organizations will at least have a default `Fund`, usually named something like "General" or "Tithe".

### `Designation`

Each `Donation` can have one or more `Designations`.  The main purpose of a designation is to say how much of a `Donation`'s money is going to a given `Fund`.  If all of a `Donation` is going to a single `Fund`, then you'll only need the one designation, but because a single `Donation` can be split between multiple funds, you'll always need to include details about a `Donation`'s `Designation`s when creating the `Donation` (more details in the [docs](https://developer.planning.center/docs/#/apps/giving/2018-08-01/vertices/donation#permissions)).

### `PaymentSource`

Every `Donation` must be associated with a `PaymentSource`. This tells Giving (and the Giving admins) what "system" originally accepted the `Donation`. In most cases, you'll want to create a `PaymentSource` for the system you're importing from and associate every `Donation` record with that `PaymentSource`.

### `Batch`

A `Batch` is almost entirely a grouping of `Donation`s. You're free to determine how to group the `Donation` records you're creating, as are each organization's administrators when creating `Batch`es in the web UI. This is the same mechanism that those admins will use to input check and cash donations manually.

When creating `Donation`'s via the API, you're required to put them in a `Batch`.

The main thing to know with `Batch`es is the idea of a "committed" `Batch`. You can start adding `Donation`s to a `Batch` right after you've created it. But in that "uncommitted" default state, those `Donation`s will not be visible to donors. This can be helpful in making sure that the data is correct before making it visible to donors. Also, while a `Donation` is in this "uncommitted" or "pending" state, you can make changes to it as needed.

After a `Batch` is committed, all of the `Donation`'s within it are also marked as "committed". At that point, they're visible to donors and any further edits made to those `Donation`s are logged and visible to Giving admins.

With all of that in mind, you can use `Batch`es in one of two ways:

1. Create an uncommitted `Batch`, add `Donation`s to it, then commit the `Batch`.
2. Create a `Batch`, commit it, then add `Donation`s to it.

In both cases, the end result is the same. The main difference in terms of _when_ the `Batch` is committed makes a difference in that route #2 does not provide you (or Giving admins) the opportunity to fix any mistakes before changes are logged and `Donation`s are made visible to donors.

Whichever route you decide to take, it's helpful to make use of the `Batch`'s `description` to help differentiate these groupings from each other and from other `Batch`es that the Giving admins might be creating on their own.

## Donors

The way that donors are associated to `Donation`s is fairly straightforward, but the process of linking and keeping track of donors in your system with those in an organization's Planning Center account, and understanding the implications of why our system is setup the way it is, will likely be one of the more challenging part of your integration.  But getting this right will go a long way to making your integration work well for your customers.

### `Person`

The `Person` object in Planning Center is so crucial that we have an entire app dedicated to managing, keeping track of, editing, and creating these records and metadata around them. Because of that, you'll need to get familiar with a few key endpoints in the [Planning Center People API](https://developer.planning.center/docs/#/apps/people) and the concepts about how we view and handle these records.

#### Searching

When you have a donation in your system (whether completed or in-process) that you want to import into Planning Center Giving, you'll likely have some understanding of who gave the gift.  Maybe you have a number of details about them, such as their name, email address, phone number, or even mailing address.  Or you may only have more limited information like the account holder's name and bank information from a scanned check. In any instance, you'll want to start by searching for a person with those same details in Planning Center People.

You can search by first and last name separately:
```
GET https://api.planningcenteronline.com/people/v2/people?where[first_name]=Gene&where[last_name]=Parm
```

As a full name:
```
GET https://api.planningcenteronline.com/people/v2/people?where[search_name]=Gene%20Parm
```

By email:
```
GET https://api.planningcenteronline.com/people/v2/people?include=emails&where[search_name_or_email]=gene@example.com
```


Or you can simply look them up by their `id`:
```
GET https://api.planningcenteronline.com/people/v2/people/52665741
```

NOTE: In the case where you already have the `Person.id` (more on getting and saving that next), you could simply use that when creating the `Donation` record, but because of a number of things that can happen to those `Person` records that are outside of your control, we highly recommend always looking up the `Person` first as a natural place to respond to the situations we'll detail below.

#### Picking a `Person`

We have a number of large organizations that use Planning Center, so there's a good chance that you'll come across a scenario where searching for a person based on first and last name will return multiple `Person` records. In this case, it's important that, unless you have a sophisticated way of further narrowing down and choosing among a list of 5 different "Peter Jones" records with a high degree of confidence, you give your user the ability to manually choose from the returned matches. We've found (in our own apps and in 3rd party integrations) that nearly any "simple" approach that lets a program make these decisions can easily result in problematic situations like one donor seeing another donor's contributions.

Within this process, it's also possible that a donor from your system won't exist yet in Planning Center. In that case, you'll need to create a new `Person` record, which can be done [via the API](https://developer.planning.center/docs/#/apps/people/2019-01-14/vertices/person#permissions), as well. 

Because these `Person` records appear in all of our Planning Center apps that these organizations use every day, it's also important to avoid "creating a new person" as the default when searching returns multiple matches. While that would be a safe option in some sense, the proliferation of new (and likely duplicate) profiles almost always creates a significant amount of extra work for each organization's administrator(s). While we give them means for sorting out those duplicates, it's much better to avoid the added burden altogether.

#### Tracking in your app

Once you've matched a donor in your system with a `Person` in Planning Center, you'll want to save the `id` of that `Person` in your own app. If you have your own database of people within your app, that could be as simple as adding a `planning_center_id` field for storing that value.  Or you may simply be saving it as a key-value store that links a line of MICR data with the `id` from Planning Center. Either way, you'll want to save that `id` to make future searches much quicker and less error-prone.

#### Person merges

The most common issue that you'll likely run into with `Person` records is that they may appear to have disappeared. One day, you'll search for a `Person`, find them, and save their `id`. And the next day, you'll try to search for them by that `id` and you'll get back a 404 in response. This could happen for one of two reasons:

1. The `Person` was deleted. If they have `Donation`s associated with them, this isn't very likely, but it is possible. In this case, you'll want to delete this `id` from your system and step back to searching for or creating the `Person` from the details you have in your system.
2. The `Person` was merged into another `Person`. This is by far the more likely case. Our People app gives administrators a means for detecting duplicate records and merging one into the other. When this happens, we keep track of the merges that have happened and give you a way to follow that trail and update your app with the new information.

At the point when you get a 404, you won't be sure whether you've hit #1 or #2. The first thing to do is to check whether this `Person` (in this case, with `id` 14994497) was merged away:

```
GET https://api.planningcenteronline.com/people/v2/person_mergers?where[person_to_remove_id]=14994497
```

If that query comes back empty, your `Person` was deleted (#1).

Example empty response:

```json
{
  "links": {
    "self": "https://api.planningcenteronline.com/people/v2/person_mergers?where[person_to_remove_id]=14994497"
  },
  "data": [],
  "included": [],
  "meta": {
    "total_count": 0,
    "count": 0,
    "can_order_by": [
      "created_at"
    ],
    "can_query_by": [
      "created_at",
      "person_to_keep_id",
      "person_to_remove_id"
    ],
    "parent": {
      "id": "127",
      "type": "Organization"
    }
  }
}
```

If it does return a record, then they were merged, and we can see which record they were merged into.

Example response when `Person` has been merged:

```json
{
  "links": {
    "self": "https://api.planningcenteronline.com/people/v2/person_mergers?where[person_to_remove_id]=14994497"
  },
  "data": [
    {
      "type": "PersonMerger",
      "id": "3518",
      "attributes": {
        "created_at": "2016-06-15T16:07:33Z",
        "person_to_keep_id": 7504309,
        "person_to_remove_id": 14994497
      },
      "relationships": {
        "person_to_keep": {
          "data": {
            "type": "Person",
            "id": "7504309"
          }
        },
        "person_to_remove": {
          "data": {
            "type": "Person",
            "id": "14994497"
          }
        }
      },
      "links": {
        "self": "https://api.planningcenteronline.com/people/v2/person_mergers/3518"
      }
    }
  ],
  "included": [],
  "meta": {
    "total_count": 1,
    "count": 1,
    "can_order_by": [
      "created_at"
    ],
    "can_query_by": [
      "created_at",
      "person_to_keep_id",
      "person_to_remove_id"
    ],
    "parent": {
      "id": "127",
      "type": "Organization"
    }
  }
}
```

In this case, `Person` 14994497 (`person_to_remove`) was merged into 7504309 (`person_to_keep`). That's great! You can update any references in your system from 14994497 to 7504309 and continue making API calls as you otherwise would. In fact, in the process of creating this `Donation` that was supposed to be associated with 14994497 you can safely associate it with 7504309 now. The result from the `PersonMerge`s endpoint says that an admin in this organization determined that 14994497 and 7504309 were the same person, so your app can simply follow suit.

#### Anonymous gifts

If the gifts as received in your system don't have any donor data at all, it's also possible to omit the `Person` relationship from the `Donation` record, which will have the effect in Planning Center Giving of showing that `Donation`'s donor as "Anonymous Donor".

