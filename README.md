# bcmsfile
Store All Entries from BCMS as Local Files

This clog explains how you can use a Bourne shell script to store all entries from the BCMS headless Content Management System (CMS) as files on the local file system. You can use this technique with any CMS.
BCMS provides a Webservice API that lets us retrieve all entries from all categories. Because that could be a large volume of data, we may need to iterate over pages of entries, which means invoking that Webservice API repeatedly – once for the first page, which indicates the number of pages, and once for each additional page. We can retrieve the subsequent pages in parallel, but at some volume, we may need to introduce throttling.

## Webservice API Paging

Quickly, an overview of how paging typically works with Webservice APIs. When retrieving all data as in this example and in other cases where data volumes could be high, Webservice APIs break data into pages of a given size and require callers to invoke the API repeatedly, passing parameters to specify page numbers, which implies that data is in a consistent order.
Typically, you call the API to retrieve the first page, which indicates the size of the page and the number of pages. You then call the API again for any pages that you want. This typically uses three parameters that can go by different names:

•	`total`: The total number of records.
•	`limit`/`pageSize`: The page size.
•	`offset`/`skip`: The number of records to skip.

## Shell Script

I have found Bash shell scripts to be efficient for prototyping, although jq syntax is a bit cumbersome (though efficient). If you think about it, the command line exposes a sort of API that provides very high-level functionality, such as grep with filename globbing, but also supports minute operations such as `sed`.

To page through all records from a Webservice API, we can use a shell script that calls itself. First, we call it with no command line arguments. In this case, it retrieves the first page, and then calls itself once for each additional page, with a command line argument that sets offset/skip appropriately.

While we’re at it, we can normalize the JSON, simplifying its structure and abstracting the underlying system (BCMS).

## This Script

The shebang line instructs the operating system to use Bash instead of Bourne, which allows use of things like ${1:0} (and advanced globbing, though that’s not relevant here).

The `start` variable contains either the first parameter passed on the command line, or the value 0 if there is no such value.

The `auth` variable stores a BCMS authentication token. By default, these are very short-lived.

The `json` variable contains a list of entries from the CMS in a format such as the following:

``` json
{
  "items": [
    {
      "_id": "6726916a6a0878c7ae2da2f9",
      "createdAt": 1730580842939,
      "updatedAt": 1730580864005,
      "templateId": "6726915f6a0878c7ae2da2f7",
      "userId": "672691516a0878c7ae2da2f3",
      "info": [
        {
          "lng": "en",
          "title": "first",
          "slug": "first"
        }
      ]
    },
    {
      "_id": "672691736a0878c7ae2da2fa",
      "createdAt": 1730580851214,
      "updatedAt": 1730584272806,
      "templateId": "672691656a0878c7ae2da2f8",
      "userId": "672691516a0878c7ae2da2f3",
      "info": [
        {
          "lng": "en",
          "title": "second",
          "slug": "second"
        }
      ]
    }
//...
  ],
  "limit": 101,
  "total": 101,
  "offset": 0
}
```

In this JSON:

- items array contains the entries.
- limit specifies the page size.
- total specifies the total number of entries.
- offset specifies the number of entries skipped before returning this list, which is zero for the first page, limit for the second page, 2*limit for the third page, and so forth.

Next, if the script is processing the first page and there are more pages, the script invokes itself once in the background for each remaining page (in parallel), passing an increasing value as the first argument on the command line to determine the number of entries to skip (offset).
