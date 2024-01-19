<!-- {% raw %} -->
<img src="https://i.imgur.com/GJZbJyx.jpg">

# Django One-to-Many Models

## Road Map

1. Setup
2. A Cat's Got to Eat!
3. Add the New `Feeding` Model
4. Make and Run the Migration
5. Create Serializer
6. Add `Feeding` to the Admin Portal
7. Adding New Feeding Functionality
8. Essential Questions
9. Lab Assignment
10. Bonus Challenge: Adding a Custom Method to Check the Feeding Status

## 1. Setup

This lesson continues to build-out Cat Collector right where the [_Django Modeles and Serializers_](https://git.generalassemb.ly/SEB-Base-Curriculum/django-models-serializers) lesson left off.

Navigate to the django-catcollector directory, enter your virtual environment ```pipenv shell```, then start the web server with `python manage.py runserver`.

## 2. A Cat's Got to Eat!

In this lesson we're going to look at how to add another model that demonstrates working with **one-to-many** relationships in Django.

Since a cat's got to eat, let's go with this relationship:

<img src="https://i.imgur.com/37yYc42.png">

<details>
<summary>
❓ How does the above relationship "read"?
</summary>
<hr>

**A Cat has many Feedings" -and- "A Feeding belongs to a Cat**

<hr>
</details>

## 3. Add the New `Feeding` Model

<details>
<summary>
❓ What module/file are we going to add the new Feeding Model in?
</summary>
<hr>

**main_app/models.py**

<hr>
</details>

### Add the `Feeding` Model

Using the ERD above as a guide, let's add the new `Feeding` Model (below the current `Cat` Model):

```python
# Add new Feeding model below Cat model
class Feeding(models.Model):
  date = models.DateField()
  meal = models.CharField(max_length=1)
```

Note that we're using a single-character to represent what meal the feeding is for:

- `B`reakfast
- `L`unch
- `D`inner

### Field.choices

Django has a [Field.choices](https://docs.djangoproject.com/en/4.1/ref/models/fields/#choices) feature that will make the single-characters more user-friendly by automatically generating a *select* dropdown in the form, using descriptions that we define.

The first step is to define a tuple of 2-tuples.  Because we also might need to access this tuple within the `Cat` class, let's define it above **both** of the Model classes:

```python
# A tuple of 2-tuples
MEALS = (
    ('B', 'Breakfast'),
    ('L', 'Lunch'),
    ('D', 'Dinner')
)
# new code above

class Cat(models.Model):
``` 

As you can see, the first item in each 2-tuple represents the value that will be stored in the database, e.g. `B`.

The second item represents the human-friendly "display" value, e.g., `Breakfast`.

Now let's enhance the `meal` field as follows:

```python
class Feeding(models.Model):
  date = models.DateField()
  meal = models.CharField(
    max_length=1,
    # add the 'choices' field option
    choices=MEALS,
    # set the default value for meal to be 'B'
    default=MEALS[0][0]
  )
```

### Add the `__str__` Method

As recommended, we should override the `__str__` method on Models so that they provide more meaningful output when they are printed:


```python
class Feeding(models.Model):
  ...

  def __str__(self):
    # Nice method for obtaining the friendly value of a Field.choice
    return f"{self.get_meal_display()} on {self.date}"
```

Check out the convenient `get_meal_display()` method Django auto-magically creates to access the human-friendly value of a Field.choice like we have on `meal`.

### Add the Foreign Key

Since a `Feeding` **belongs to** a `Cat`, it must hold the `id` of the cat object it belongs to --- yep, it needs a **foreign key**!

Here's how it's done - "Django-style":

```python
class Feeding(models.Model):
  date = models.DateField()
  meal = models.CharField(
    max_length=1,
	 choices=MEALS,
	 default=MEALS[0][0]
  )
  # Create a cat_id FK
  cat = models.ForeignKey(Cat, on_delete=models.CASCADE)

  def __str__(self):
    return f"{self.get_meal_display()} on {self.date}"
```

As you can see, the `ForeignKey` field-type is used to create a *one-to-many* relationship.

> 👀 In the database, the column in the `feedings` table for the FK will actually be called `cat_id` because Django by default appends `_id` to the name of the attribute we use in the Model.

The first argument provides the parent Model.

In a one-to-many relationship, the `on_delete=models.CASCADE` is required.  It ensures that if a Cat record is deleted, all of the child Feedings will be deleted automatically as well - thus avoiding _orphan_ records (seriously, that's what they're called).

## 4. Make and Run the Migration

We added/updated a new Model, so it's that time again...

```
python manage.py makemigrations
```

<details>
<summary>
❓ Now what command do we need to run?
</summary>
<hr>

```
python manage.py migrate
```

<hr>
</details>

After creating a Model, especially one that relates to another, it's always a good idea to test drive the Model and relationships in the shell.

## 5. Create Serializer

Next, we'll set up our feeding serializer in the serializers.py file:

```python
class FeedingSerializer(serializers.ModelSerializer):
  class Meta:
    model = Feeding
    fields = '__all__'
    read_only_fields = ('cat',)
```

We have configured the `cat` field as `read_only=True` to eliminate the necessity of including the `cat_id` in the body of a POST request during the creation of a feeding. Instead, this value will be derived from the URL parameter, a method we are yet to implement. This approach streamlines the process and ensures a more intuitive and cleaner data submission.

## 6. Add `Feeding` to the Admin Portal

Remember, before a Model can be CRUD'd using the built-in Django admin portal, the Model must be registered.

Update `main_app/admin.py` so that it looks as follows:

```python
from django.contrib import admin
# add Feeding to the import
from .models import Cat, Feeding

admin.site.register(Cat)
# register the new Feeding Model 
admin.site.register(Feeding)
```

Now browse to `localhost:8000/admin` and click on the `Feeding` link.

<img src="https://i.imgur.com/S1w9o0N.png">

Check out those select drop-downs for assigning both the Meal and the Cat!

### Custom Field Labels

By default, the admin portal uses the attribute names as labels with the first character capitalized.  For example:  **Date**.

Perhaps you would like to override a label?

Because Django subscribes to the philosophy that **a Model is the single, definitive source of truth about your data**, you can bet that's where we will add customization.

In `main_app/models.py`, add the desired user-friendly label to the field-type like so:

```python
class Feeding(models.Model):
  # the first optional positional argument overrides the label
  date = models.DateField('Feeding Date')
  ...
```

Refresh and...

<img src="https://i.imgur.com/KMKlEgA.png">

## 7. Adding New Feeding Functionality

Now we want to add full CRUD capability to a feeding model resource.

As we did previously, we will use class-based views to productively perform create, read, update, and delete data operations for a Model.

There's several steps we're going to need to complete, so let's get started...

### Add Another `path(...)` to `urls.py`

Every `Feeding` object needs a value for its `cat_id` foreign key that holds the primary key of the cat object that it belongs to.

Therefore, we need to ensure that the route includes a URL parameter for capturing the cat's `id` like we've done in other routes:

```python
# main_app/urls.py

urlpatterns = [
	...
	path('cats/<int:cat_id>/feedings/', FeedingListCreate.as_view(), name='feeding-list-create'),
	path('cats/<int:cat_id>/feedings/<int:id>/', FeedingDetail.as_view(), name='feeding-detail'),
]
```

Don't forget to import the views FeedingListCreate and FeedingDetail - they don't exist yet but we will create them now.

### Generating our View Logic

The `FeedingListCreate` class, a Class-based View (CBV), represents a nuanced implementation in comparison to our `CatList` view. Both CBVs extend from the REST framework's `ListCreateAPIView` generics, but they serve distinct purposes and exhibit differing behaviors in data retrieval and handling.

```python
class FeedingListCreate(generics.ListCreateAPIView):
  serializer_class = FeedingSerializer

  def get_queryset(self):
    cat_id = self.kwargs['cat_id']
    return Feeding.objects.filter(cat_id=cat_id)

  def perform_create(self, serializer):
    cat_id = self.kwargs['cat_id']
    cat = Cat.objects.get(id=cat_id)
    serializer.save(cat=cat)
```

In the case of the `CatList` view, we employ a straightforward approach by setting the queryset to `Cat.objects.all()`. This configuration is apt for scenarios where the goal is to retrieve a comprehensive list of all cat instances, without applying any specific filters.

However, the `FeedingListCreate` view necessitates a more tailored approach to data retrieval. The objective here is to display feedings that are associated with a specific cat, identified by the `cat_id` present in the URL. To accommodate this requirement, we override the `get_queryset` method. This alteration ensures that the queryset dynamically adapts to include only those feeding instances that correspond to the specified `cat_id`.

Moreover, the introduction of the `perform_create` method adds another layer of specificity. This method is pivotal during the creation of new feeding instances. It meticulously extracts the `cat_id` from the URL parameters and fetches the corresponding `Cat` instance. Subsequently, it associates this instance with the new feeding record being created. This method ensures the correct association between related entities, in this context, ensuring that feedings are accurately linked to their respective cats.

This design pattern is common in Django REST framework when you need to provide different querysets based on the request's context or parameters.

And now for the remaining view:

```python
class FeedingDetail(generics.RetrieveUpdateDestroyAPIView):
  serializer_class = FeedingSerializer
  lookup_field = 'id'

  def get_queryset(self):
    cat_id = self.kwargs['cat_id']
    return Feeding.objects.filter(cat_id=cat_id)
```

By now the code above should slowly start to make a bit more sense - the more reps the better we solidy our understanding.

Don't forget to import the Feeding model and serializer!

> 💡 This is a good place to stop and test the new endpoints on Postman.

### Ordering Feedings

It would be nice to have the most recent dates displayed at the top.

Yup, _a Model is the single, definitive source of truth..._

The answer lies in adding a `class Meta` with an `ordering` class attribute within the `Feeding` Model:

```python
class Feeding(models.Model):
  ...
  def __str__(self):
    # Nice method for obtaining the friendly value of a Field.choice
    return f"{self.get_meal_display()} on {self.date}"

  # change the default sort
  class Meta:
    ordering = ['-date']
```

Feed the cats!

Be sure to check out the BONUS section on your own.

## 8. ❓ Essential Questions (2 mins)

<details>
<summary>
(1) When two Models have a one-many relationship, which Model must include a field of type <code>models.ForeignKey</code>, the one side, or the many side?
</summary>
<hr>

**The many ("belongs to") side.**

<hr>
</details>

Assuming a `Author --< Book` relationship, answer the following three questions...

<details>
<summary>
(2) Which Model gets the <code>ForeignKey</code> field and what's a good name for that attribute?
</summary>
<hr>

The `Book` Model with an attribute of `author`, i.e.:

```python
author = models.ForeignKey(Author, on_delete=models.CASCADE)
```

<hr>
</details>

<details>
<summary>
(3) How would we access a <code>book</code> object's author object?
</summary>
<hr>

```python
book.author
```

<hr>
</details>

## 9. Lab Assignment

As usual, the lab calls for implementing the same features in your Finch Collector project.

## 10. Bonus Challenge: Adding a Custom Method to Check the Feeding Status

When adding _business logic_ to an application, always consider adding that logic to the Models first.

This approach is referred to as "fat models / skinny views" and results in keeping code DRY.

We're going to add a custom method to the `Cat` Model that help's notify us whether or not the cat has had at least the number of feedings for the current day as there are MEALS.

### The `fed_for_today` Custom Model Method

Let's add a one line method to the `Cat` Model class:

```python
from django.db import models
# add this import
from datetime import date

class Cat(models.Model):
  ...
  # add this new method
  def fed_for_today(self):
    return self.feeding_set.filter(date=date.today()).count() >= len(MEALS)
```

Be sure to add the import at the top of _models.py_.

The `fed_for_today` method demonstrates the use of `filter()` to obtain a `<QuerySet>` for today's feedings, then `count()` is chained on to the query to return the actual number of objects returned.

Considering we are building a RESTful API, we should include the result of the fed_for_today method in our JSON response when querying Cat instances. In order to do that, we need to modify our **CatSerializer** to include this information. Here's how we do that:

```python
class CatSerializer(serializers.ModelSerializer):
  fed_for_today = serializers.SerializerMethodField() # add this line

  class Meta:
    model = Cat
    fields = '__all__'
  
  # add method below
  def get_fed_for_today(self, obj):
    return obj.fed_for_today()

```

In this setup:

- fed_for_today is a custom field added to CatSerializer.
- get_fed_for_today takes the Cat instance (obj) as an argument and returns the result of the fed_for_today method on that instance.

When you serialize a Cat instance using this modified CatSerializer, it will include the fed_for_today field in the JSON response, which will be either True or False depending on whether the cat has been fed for all meal times on the current day.

Congrats on using a custom Model method to implement business logic!

## Resources

[Django Model API](https://docs.djangoproject.com/en/4.1/ref/models/)

<!-- {% endraw %} -->
