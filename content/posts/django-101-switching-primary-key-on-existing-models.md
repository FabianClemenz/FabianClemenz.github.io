+++ 
draft = false
date = 2023-03-09T13:31:46+01:00
title = "django 101: switching the primary key on existing models"
description = "This Django 101 takes you through the steps of switching from one primary key to another on existing models, without harming your data"
slug = "django-101-switching-primary-key-on-existing-models"
authors = ["Fabian Clemenz"]
tags = ["django", "database", "primary key switch", "uuid"]
categories = ["101", "programming"]
externalLink = ""
series = ["django 101"]
+++
#### *or: living on the edge of data loss*

A while ago, we got the task to switch from a 22 long char field to our internal uuid as primary key. The 22-character long id was the primary key from our source system. The idea was to decouple both systems, so we can create virtual entries, without providing a random char, which could have led to errors in the future. Our database table contained over 3.4 million entries at this time. Linked to round about 16 other tables in foreign key and many to many relations.

After research, I used the instructions found in this [stackoverflow answer](https://stackoverflow.com/questions/48200026/how-to-make-uuid-field-default-when-theres-already-id-field/48235821#48235821), which I updated afterwards to fix what I noticed.

## prerequisites

### caution

Before applying these steps on your production database - **ALWAYS** test it with test data **AND** backup your database. And at best - test it several times. That saved me a lot of work, because I needed three iterations for all migrations to work as expected.

### sample code

I created a sample django project to show the single steps. It can be found in my [Github repository](https://github.com/FabianClemenz/primary-key-switch).

The `0000_base_db_dump_before_pk_switch.json` in the fixtures folder contains a base dataset on which we will perform our operations. In `0001_db_dump_after_pk_switch.json` you can find the end result.

I will reference the migrations located in [primary_key_switch / test_app / migrations](https://github.com/FabianClemenz/primary-key-switch/tree/draft/initial/primary_key_switch/test_app/migrations) while I explain the single steps.

### basic models

In django, without providing a specific primary key, it will automatically generate an integer based primary key.

```python
class TestModel(models.Model):
    name = models.CharField(max_length=200, verbose_name='Name')

    class Meta:
        verbose_name = 'Test Model'

class TestForeignKeyModel(models.Model):
    test_model = models.ForeignKey(TestModel, on_delete=models.CASCADE, related_name='test_model')

    class Meta:
        verbose_name = 'Test ForeignKey Model'

class TestManyToManyModel(models.Model):
    test_models = models.ManyToManyField(TestModel, related_name='test_manytomany_models')

    class Meta: 
        verbose_name = 'Test ManyToMany Model'
```

## step by step

### step 1: add an uuid field

The first step is - who would have known - adding the uuid field to our wanted model (`TestModel`) but accepting null values here (`0002_testmodel_uuid.py`). After that fill unique uuids (`0003_set_unique_uuids.py`) and reset the field to be unique and not nullable (`0004_set_unique_true_on_uuid.py`).

```python
class TestModel(models.Model):
    uuid = models.UUIDField(default=uuid.uuid4, unique=True)
    name = models.CharField(max_length=200, verbose_name='Name')

    class Meta:
        verbose_name = 'Test Model'
```

### step 2: add new foreign keys to other models

Now we need to link our `TestModel` again with the other models through a foreign key relation, but this time through the uuid field. This field must be nullable in the first step (`0005_testforeighkeymodel_test_model_uuid.py`), so we can link the correct instance by hand (`0006_uuid_fk_test_foreign_key_model.py`). I would recommend using a _uuid suffix to distinguish between the fields.

```python
class TestForeignKeyModel(models.Model):
    test_model = models.ForeignKey(TestModel, on_delete=models.CASCADE, related_name='test_model')
    test_model_uuid = models.ForeignKey(TestModel, null=True, to_field='uuid', on_delete=models.CASCADE, related_name='test_model_uuid')

    class Meta:
        verbose_name = 'Test ForeignKey Model'
```

### step 3: add new model for many to many relation

Changing many to many relations differs from changing foreign key relations. We need to create a temporary model which will be used as a table for the new relation (`0007_testthroughmodel_and_more.py`). This is called a through model or a through table. And as seen before, copy the wanted data to the new table (`0008_uuid_m2m_testmanytomanymodel.py`).

```python
class TestThroughModel(models.Model):
    test_many_to_many_model = models.ForeignKey(TestManyToManyModel, null=True, on_delete=models.CASCADE)
    test_model = models.ForeignKey(TestModel, null=True, to_field='uuid', on_delete=models.CASCADE)

    class Meta:
        verbose_name = 'Test Through Model'

class TestManyToManyModel(models.Model):
    test_models = models.ManyToManyField(TestModel, related_name='test_manytomany_models')
    test_models_uuid = models.ManyToManyField(TestModel, through='TestThroughModel', related_name='test_manytomany_models_uuid')

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

### step 4: delete old connections

Now it’s time to remove the old connections between our models. We just remove the many to many and foreign key fields without the _uuid suffix. Also we don’t delete our `TestThroughModel` yet (`0009_remove_test_foreignkeymodel_test_model_and_more.py`).

```python
class TestModel(models.Model):
    uuid = models.UUIDField(default=uuid.uuid4, unique=True)
    name = models.CharField(max_length=200, verbose_name='Name')

    class Meta:
        verbose_name = 'Test Model'

class TestForeignKeyModel(models.Model):
    test_model_uuid = models.ForeignKey(TestModel, null=True, to_field='uuid', on_delete=models.CASCADE, related_name='test_model_uuid')

    class Meta:
        verbose_name = 'Test ForeignKey Model’

class TestThroughModel(models.Model):
    test_many_to_many_model = models.ForeignKey(TestManyToManyModel, null=True, on_delete=models.CASCADE)
    test_model = models.ForeignKey(TestModel, null=True, to_field='uuid', on_delete=models.CASCADE)

    class Meta:
        verbose_name = 'Test Through Model'

class TestManyToManyModel(models.Model):
    test_models_uuid = models.ManyToManyField(TestModel, through='TestThroughModel', related_name='test_manytomany_models_uuid')

    class Meta:
        verbose_name = 'Test ManyToMany Model'

```

### step 5: delete the db constraints on the new foreign key fields

This is a crucial step, because django will connect through the unique constraints of the uuid which will be dropped, after we change our primary key to the new uuid field (`0010_alter_testforeignkeymodel_test_model_uuid_and_more.py`).

```python
class TestForeignKeyModel(models.Model):
    test_model__uuid = models.ForeignKey(TestModel, db_constraint=False, null=True, to_field='uuid', on_delete=models.CASCADE, related_name='test_model_uuid')

    class Meta:
        verbose_name = 'Test ForeignKey Model'

class TestThroughModel(models.Model):
    test_many_to_many_model = models.ForeignKey(TestManyToManyModel, null=True, on_delete=models.CASCADE)
    test_model = models.ForeignKey(TestModel, db_constraint=False, null=True, to_field='uuid', on_delete=models.CASCADE)

    class Meta:
        verbose_name = 'Test Through Model'
```

### step 6: set new primary key field on TestModel

Now it’s time to set our uuid field as the primary key. We can also remove the to_field declarative in our foreign key relations (`0011_remove_testmodel_id_and_more.py`).

```python
class TestModel(models.Model):
    uuid = models.UUIDField(primary_key=True, default=uuid.uuid4, unique=True)
    name = models.CharField(max_length=200, verbose_name='Name')

    class Meta:
        verbose_name = 'Test Model'

class TestForeignKeyModel(models.Model):
    test_model_uuid = models.ForeignKey(TestModel, db_constraint=False, null=True, on_delete=models.CASCADE, related_name='test_model_uuid')

    class Meta:
        verbose_name = 'Test ForeignKey Model'

class TestThroughModel(models.Model):
    test_many_to_many_model = models.ForeignKey(TestManyToManyModel, null=True, on_delete=models.CASCADE)
    test_model = models.ForeignKey(TestModel, db_constraint=False, null=True, on_delete=models.CASCADE)

    class Meta:
        verbose_name = 'Test Through Model'
```

### step 7: rename fields and remove _uuid suffix

We rename the fields and remove the _uuid suffix, so our old code will work as before (`0012_rename_test_model_uuid_testforeignkeymoel_test_model_and_more.py`). It is crucial here to be sure that django generates **rename migrations** instead of deleting the old field and adding a new one. This will delete your data! So check the generated migrations before applying them.

```python
class TestForeignKeyModel(models.Model):
    test_model = models.ForeignKey(TestModel, db_constraint=False, null=True, on_delete=models.CASCADE, related_name='test_model_uuid')

    class Meta:
        verbose_name = 'Test ForeignKey Model'

class TestManyToManyModel(models.Model):
    test_models = models.ManyToManyField(TestModel, through='TestThroughModel', related_name='test_manytomany_models_uuid')

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

### step 8: remove unnecessary options and change related name

This step should not be done with the previous step, to be sure that django only renames the fields. We will now change the related names and remove other options such as `db_constraint` (`0013_alter_test_foreignkeymodel_test_model_and_more.py`).

```python
class TestForeignKeyModel(models.Model):
    test_model_uuid = models.ForeignKey(TestModel, on_delete=models.CASCADE, related_name='test_model')

    class Meta:
        verbose_name = 'Test ForeignKey Model'

class TestThroughModel(models.Model):
    test_many_to_many_model = models.ForeignKey(TestManyToManyModel, on_delete=models.CASCADE)
    test_model = models.ForeignKey(TestModelon_delete=models.CASCADE)

    class Meta:
        verbose_name = 'Test Through Model'

class TestManyToManyModel(models.Model):
    test_models = models.ManyToManyField(TestModel, through='TestThroughModel', related_name='test_manytomany_models')

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

And that's it. We changed the primary key from one field to another field. This can be done with fields other than uuid and the automatic id field in django.

---

## miscellaneous

If you don’t want to use a through model you need to do the following 5 extra steps. These steps could be included in the migrations before, but the goal here was to explicitly show the needed steps and explain them with their pitfalls.

### extra step 1: add the “old” many to many field

We need to add the **"old"** many to many field, so we can copy the data back (`0014_testmanytomanymodel_test_models_old.py`).

```python
class TestManyToManyModel(models.Model):
    test_models = models.ManyToManyField(TestModel, through='TestThroughModel', related_name='test_manytomany_models')
      test_models_old = models.ManyToManyField(TestModel, related_name='test_manytomany_models_old'

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

### extra step 2: copy the values - again

Now - *again* - we copy the values from one M2M to another M2M (`0015.copy_m2m.py`).

### extra step 3: delete the through model and many to many field

We then delete the `TestThroughModel` and the related M2M field (`0016_remove_testmanytomanymodel_test_models_and_more.py`).

```python
class TestManyToManyModel(models.Model):
      test_models_old = models.ManyToManyField(TestModel, related_name='test_manytomany_models_old'

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

### extra step 4: rename the many to many field

Now - *as before* - rename the many to many field. And - *again* - be careful, that django only generates a rename migration (`0017_rename_test_models_old_testmanytomanymodel_test_models.py`).

```python
class TestManyToManyModel(models.Model):
      test_models = models.ManyToManyField(TestModel, related_name='test_manytomany_models_old'

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

### extra step 5: change back related_name

Last but not least - change back the `related_name` (`0018_alter_testmanytomanymodel_test_models.py`). Now we have the same structure as in the beginning, but with a uuid as primary key.

```python
class TestManyToManyModel(models.Model):
      test_models = models.ManyToManyField(TestModel, related_name='test_manytomany_models'

    class Meta:
        verbose_name = 'Test ManyToMany Model'
```

## code after all our changes

```python
class TestModel(models.Model):
    uuid = models.UUIDField(primary_key=True, default=uuid.uuid4, unique=True)
    name = models.CharField(max_length=200, verbose_name='Name')

    class Meta:
        verbose_name = 'Test Model'

class TestForeignKeyModel(models.Model):
    test_model = models.ForeignKey(TestModel, on_delete=models.CASCADE, related_name='test_model')

    class Meta:
        verbose_name = 'Test ForeignKey Model'

class TestManyToManyModel(models.Model):
    test_models = models.ManyToManyField(TestModel, related_name='test_manytomany_models')

    class Meta: 
        verbose_name = 'Test ManyToMany Model'
```

## conclusion

It can be really tricky changing primary key on grown databases but with a good plan, stackoverflow and a lot of trial and error on test data, you can create migrations without harming your production environment.
