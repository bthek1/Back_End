# Datatable Serverside Processing

Server-side processing means **DataTables delegates filtering, sorting, and pagination to your backend**, sending these parameters via AJAX. DRF then handles these to return a paginated, filtered, and sorted response.

Author

Benedict Thekkel

## 🗂️ 1. DataTables Frontend Setup

### Example JS:

``` js
$('#clientTable').DataTable({
  serverSide: true,
  processing: true,
  ajax: {
    url: '/api/clients/',
    type: 'GET',
  },
  columns: [
    { data: 'first_name' },
    { data: 'last_name' },
    { data: 'treatment_name' },
    { data: 'appointments_upcoming' },
    // ...
  ]
});
```

> Ensure `data` matches the field names in your serializer.

------------------------------------------------------------------------

## 🔁 2. What DataTables Sends to the Backend

When serverSide mode is on, it sends:

| Param                    | Example          | Meaning                      |
|--------------------------|------------------|------------------------------|
| `draw`                   | `1`              | Draw counter                 |
| `start`                  | `0`              | Offset for pagination        |
| `length`                 | `10`             | Page size                    |
| `search[value]`          | `john`           | Global search value          |
| `order[0][column]`       | `2`              | Column index to sort on      |
| `order[0][dir]`          | `asc` or `desc`  | Sorting direction            |
| `columns[i][data]`       | `treatment_name` | Field name for column `i`    |
| `columns[i][searchable]` | `true`           | Can this column be searched? |

------------------------------------------------------------------------

## ⚙️ 3. Backend: Parse and Respond to DataTables

### 📦 Custom DRF Filter Backend

``` python
from rest_framework.filters import BaseFilterBackend
from django.db.models import Q
import operator
from functools import reduce

class CustomDatatablesFilterBackend(BaseFilterBackend):
    def filter_queryset(self, request, queryset, view):
        draw = int(request.GET.get('draw', 1))
        start = int(request.GET.get('start', 0))
        length = int(request.GET.get('length', 10))

        # Global search
        search_value = request.GET.get('search[value]')
        if search_value:
            fields = [col['name'][0] for col in view.get_datatables_fields()]
            q = reduce(operator.or_, [Q(**{f"{field}__icontains": search_value}) for field in fields], Q())
            queryset = queryset.filter(q)

        # Ordering
        order_col_index = request.GET.get('order[0][column]')
        order_dir = request.GET.get('order[0][dir]')
        if order_col_index is not None:
            col_name = request.GET.get(f'columns[{order_col_index}][data]')
            if col_name:
                order = f"-{col_name}" if order_dir == 'desc' else col_name
                queryset = queryset.order_by(order)

        # Pagination
        return queryset[start:start+length]
```

------------------------------------------------------------------------

## 🧩 4. Serializer

Support computed/annotated fields:

``` python
class ClientListSerializer(serializers.ModelSerializer):
    appointments_upcoming = serializers.SerializerMethodField()
    treatment_name = serializers.SerializerMethodField()

    class Meta:
        model = Client
        fields = ['first_name', 'last_name', 'appointments_upcoming', 'treatment_name']

    def get_appointments_upcoming(self, obj):
        return getattr(obj, 'appointments_upcoming', 0)

    def get_treatment_name(self, obj):
        return getattr(obj, 'treatment_name', '')
```

------------------------------------------------------------------------

## 🧮 5. ViewSet with Annotated QuerySet

``` python
class ClientViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = ClientListSerializer
    filter_backends = [CustomDatatablesFilterBackend]

    def get_queryset(self):
        now = timezone.now()
        latest_treatment = Treatment.objects.filter(client=OuterRef("pk")).order_by("-start_date")

        return Client.objects.annotate(
            treatment_name=Subquery(latest_treatment.values("name")[:1]),
            appointments_upcoming=Count(
                "appointments", filter=Q(appointments__when__gte=now)
            )
        )
    
    def get_datatables_fields(self):
        return [
            {"name": ["first_name"]},
            {"name": ["last_name"]},
            {"name": ["treatment_name"]},
            {"name": ["appointments_upcoming"]},
        ]
```

------------------------------------------------------------------------

## 📊 6. Return the Right Response Shape

DataTables expects:

``` json
{
  "draw": 1,
  "recordsTotal": 100,
  "recordsFiltered": 25,
  "data": [{...}, {...}, ...]
}
```

If using DRF pagination, override `paginate_queryset()` and `get_paginated_response()` in a custom pagination class to return this shape.

------------------------------------------------------------------------

## 🛠️ 7. Optional Enhancements

| Feature | How to Implement |
|----|----|
| Column-specific search | Parse `columns[i][search][value]` |
| Multi-column sort | Parse all `order[i]` fields |
| Date range filters | Use query params like `start_date_from`, `start_date_to` |
| Fuzzy search | Use `__icontains`, `__regex`, or trigram similarity |
| Permissions | Restrict queryset with `.for_user(user)` or custom managers |
| Large datasets | Use `values()` or `only()` to reduce query cost |

------------------------------------------------------------------------

## 🧠 Tips

- Use `.annotate()` for computed fields.
- Avoid N+1 queries by using `select_related()` and `prefetch_related()`.
- Use `Subquery()` + `OuterRef()` to include related model fields efficiently.
- Validate fields used for filtering and sorting to avoid injection attacks.

------------------------------------------------------------------------

## 🧪 Testing & Debugging

Enable logging of SQL queries to debug performance:

``` python
import logging
logger = logging.getLogger('django.db.backends')
logger.setLevel(logging.DEBUG)
logger.addHandler(logging.StreamHandler())
```

------------------------------------------------------------------------

Back to top
