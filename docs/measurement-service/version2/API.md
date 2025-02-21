# Version2 Measurement API Documents

- BaseUrl: https://carencoinc.com/api/v2/measurement

## GetFootprints

### Endpoint

| Method | URL                          |
|--------|------------------------------|
| POST   | `/users/{userId}/footprints` |

### Request

#### Parameters(@RequestParam)

| Name      | Type          | Description                                   | Required | Remarks                                                             |
|-----------|---------------|-----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String        | API version information (format: YYYY-MM-DD)  | No       | If not provided, the latest API version will be used automatically. |
| `from`    | LocalDateTime | Query start date and time                     | No       |                                                                     |
| `to`      | LocalDateTime | Query end date and time                       | No       |                                                                     |
| `size`    | int           | Number of records to retrieve                 | No       |                                                                     |
| `page`    | int           | Page number of the queried data               | No       |                                                                     |
| `sort`    | String        | Sorting method (e.g., measuredDateTime, desc) | No       |                                                                     |

> ### Additional Query Logic for `GetFootprintRecords`
> The `findByUserIdAndDateTimeRange` method includes logic to filter records based on various conditions. Below are the
> details:
>
> #### Query Conditions
>
> 1. **Both `from` and `to` parameters are provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is between `fromDateTime` and
     `toDateTime` (inclusive).
     >     - Repository method: `findByUserIdAndMeasuredDateTimeBetween`.
>
> 2. **Only `from` parameter is provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is after `fromDateTime`.
     >     - Repository method: `findByUserIdAndMeasuredDateTimeAfter`.
>
> 3. **Only `to` parameter is provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is before `toDateTime`.
     >     - Repository method: `findByUserIdAndMeasuredDateTimeBefore`.
>
> 4. **Neither `from` nor `to` parameters are provided:**
     >     - Returns all records for the given user, without date filtering.
     >     - Repository method: `findByUserId`.
>
> #### Pagination and Sorting
>
> - **Pagination:**
    >     - The `page` and `size` parameters determine the pagination behavior.
    >     - These are passed into the `PageRequest` object to fetch the corresponding page of records.
>
> - **Sorting:**
    >     - The `sort` parameter defines the sorting behavior. It should follow the format: `field,direction`.
    >         - `field`: The name of the field to sort by (e.g., `measuredDateTime`).
    >         - `direction`: Sorting direction (`asc` for ascending, `desc` for descending). Defaults to ascending if
    omitted.
    >     - Example values:
    >         - `measuredDateTime,desc`: Sort by `measuredDateTime` in descending order.
    >         - `weight,asc`: Sort by `weight` in ascending order.
    >
- **Error Handling:**
  >         - If the `sort` parameter is invalid, an `IllegalArgumentException` is thrown with a message explaining the
  expected format.
>
> #### Example Query Scenarios
>
> 1. **Retrieve all records for a user within a specific date range, sorted by timestamp in descending order:**
     >     - Parameters: `from=2025-01-01T00:00:00`, `to=2025-01-31T23:59:59`, `sort=measuredDateTime,desc`.
>
> 2. **Retrieve all records after a specific date:**
     >     - Parameters: `from=2025-01-01T00:00:00`, `sort=measuredDateTime,asc`.
>
> 3. **Retrieve paginated records without any date filters:**
     >     - Parameters: `page=1`, `size=10`.

### Response

#### Body

| Name                      | Type          | Description                                                       |
|---------------------------|---------------|-------------------------------------------------------------------|
| `success`                 | boolean       | Indicates whether the API call was successful (true/false)        |
| `data`                    | Object        | Contains data related to the user's plantar pressure              |
| `data.id`                 | UUID          | Unique identifier (ID) for the plantar pressure record            |
| `data.userId`             | String        | ID of the user who requested the measurement                      |
| `data.measuredDateTime`   | LocalDateTime | The date and time when the plantar pressure measurement was taken |
| `data.firstClassType`     | Double        | The first-ranked plantar pressure class type                      |
| `data.firstAccuracy`      | Double        | The accuracy (similarity) of the first-ranked class type          |
| `data.secondaryClassType` | Double        | The second-ranked plantar pressure class type                     |
| `data.secondaryAccuracy`  | Double        | The accuracy (similarity) of the second-ranked class type         |
| `data.thirdClassType`     | Double        | The third-ranked plantar pressure class type                      |
| `data.thirdAccuracy`      | Double        | The accuracy (similarity) of the third-ranked class type          |
| `data.leftFootLength`     | Double        | Length of the left foot (in millimeters)                          |
| `data.leftFootWidth`      | Double        | Width of the left foot (in millimeters)                           |
| `data.rightFootLength`    | Double        | Length of the right foot (in millimeters)                         |
| `data.rightFootWidth`     | Double        | Width of the right foot (in millimeters)                          |
| `data.footprintImageUrl`  | String        | URL of the saved plantar pressure image                           |
| `data.weight`             | Double        | The user's weight (in kilograms)                                  |

---
