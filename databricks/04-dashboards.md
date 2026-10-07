# Databricks Dashboards & Visualization

Building dashboards on top of SQL datasets: parameters and filters, map visuals, refresh schedules, sharing, and alerts.

---

## 1. What a dashboard is for

An **analytics application** is the interface between different user groups and the data they want insights from. It can take several forms: a dashboard, a web application, a data stream, and so on.

---

## 2. Building a dashboard

1. **Data tab**: add a dataset with _+ Add SQL dataset_ (or _+ Add dataset_).
2. **Page tab**: add a title (use `#` for a heading), visuals, and a global filter widget.

### Using parameters in queries

Parameters let viewers change a query's input from the dashboard instead of editing the SQL.

1. In the **Data tab**, compare a field against a parameter in the `WHERE` clause, using `LIKE concat('%', :parameter_name, '%')` for a partial match, then run the query.

```sql
SELECT i.*, e.agent_name
FROM <catalog>.default.insurance AS i
LEFT JOIN <catalog>.default.employee AS e
USING (agent_id)
WHERE e.agent_name LIKE concat('%', :agentName, '%');
```

2. In the **Page tab**, add a new filter widget and choose it from _Parameters_.
<img width="1932" height="870" alt="image" src="https://github.com/user-attachments/assets/51bd9c31-d919-42e9-94af-2ee28a10fa99" />


For a simple **comparison**, the same idea works with `field < :parameter_name`. Enter a value for the parameter, press _Enter_, then refresh the dashboard to apply it.
<img width="1718" height="728" alt="image" src="https://github.com/user-attachments/assets/678114d7-65ca-4cb3-b933-2ca0a5597e76" />


---

## 3. Visualization options

### Map visualizations

| Type           | How it works                                                    | Best for                                             |
| -------------- | --------------------------------------------------------------- | ---------------------------------------------------- |
| Choropleth map | color gradients represent data density or values across regions | comparing data across regions                        |
| Marker map     | markers represent individual data points                        | precise, location-based insight into specific places |

### Number formatting

`0a`: displays numbers in the thousands and above in short form (eg, `5k`).
<img width="2042" height="658" alt="image" src="https://github.com/user-attachments/assets/da5799e1-dc48-4c49-b9c5-14adb67def16" />

Databricks documents the full set of numeric format options in its [visualization docs](https://learn.microsoft.com/en-us/azure/databricks/visualizations/format-numeric-types). 

---

## 4. Refreshing a dashboard

Good practice:

- Set automatic refresh intervals, but balance the frequency to avoid lag.
- Refresh only the relevant sections.
- Watch for system performance issues, and avoid piling up manual updates.

**Steps**

1. **Publish** the dashboard. It lets you enable scheduled refreshes and grant access to other users. Then switch to the _Published_ view.
2. Click _Refresh_, or set an automatic refresh with _Schedule_ and pick the interval.
  <img width="1768" height="723" alt="image" src="https://github.com/user-attachments/assets/3903269d-e240-45d8-a588-faeb37af222c" />
3. Once a schedule exists, the button changes to _Subscribe_. From there you can edit the schedule and manage subscribers, who will receive email updates of the dashboard.

---

## 5. Dashboard management

|Action|Purpose|
|---|---|
|Clone|duplicate a dashboard to experiment safely, which also keeps previous iterations intact as a basic version history|
|Export|save in various formats (JSON by default) for sharing or offline use|
|Delete|reduces clutter and keeps the workspace organized|

In the _Draft_ view you can choose to clone, export, or move a dashboard to trash. To restore or permanently delete it, go to _Workspace_ then _Trash_.

### Sharing (via URL) and permissions

Permission levels:
<img width="1424" height="610" alt="image" src="https://github.com/user-attachments/assets/1f18ef12-158a-4217-8393-23d89eeb63cd" />


Permissions are inherited from the enclosing folder, so anyone with folder access can collaborate on the dashboards inside it.

### Alerts

Databricks SQL alerts run a query on a schedule, evaluate a defined condition, and send a notification when a threshold is met. 

Notifications can be **customized** for specific users or groups, which keeps the right people informed about important changes in the data.
<img width="1986" height="892" alt="image" src="https://github.com/user-attachments/assets/6a009669-9dbc-4f00-84fe-9882f5344f23" />


---

## Quick reference: which do I need?

- **Let viewers change a query's input without editing SQL**: parameters + filter widget
- **Compare values across regions**: choropleth map
- **Show specific locations**: marker map
- **Keep a dashboard up to date automatically**: publish, then schedule a refresh
- **Try changes without risking the live version**: clone
- **Hand the dashboard to someone else**: share by URL (check folder permissions)
- **Get notified when a metric crosses a threshold**: SQL alert

---

_Part of my [Databricks notes](https://claude.ai/chat/README.md), written while upskilling in data analytics._
