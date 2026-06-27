---
layout: post
title:  "Full-text Fuzzy Search with pg_trgm and Triggers"
date:   2015-12-05 19:21:55
categories: postgres triggers rails
---

SQL triggers is something that everybody heard a lot about. Before start we need to talk about benefits and burdens.

### Pros

1. It's very fast. If you are using any kind of adapter you pay for all the wrappers and connectors. With triggers, all operations are very cheap because they are handled by PostgreSQL directly.
2. It works like magic. There is no code in your repository, no callbacks on your models.

### Cons

1. It works like a magick. You can't easy determine all operations. All this `\df+` stuff and so on.
2. It's hard to change. Really if you want change trigger you need to drop it and hang it again. With full trigger code. You can use [hair_trigger](<https://github.com/jenseng/hair_trigger>) to simplify it but anyway.
3. It increases requirements on your developers. They should know SQL. In perfect world your team is awesome. But it's business.
4. It will cause some problems in cause if you are using STI because you can't define model by it's tablename.
5. It can cause problems on backups restoring. On restoring large tables somebody can use `--disable-triggers` to speed up process and it's not that you are expecting.

If you are still want it let's continue. You have `MobilePhone`, `Laptop` and `Ad`. `Ad` contains info about `Product` (in this case `MobilePhone` or `Laptop`). You want to provide some kind of faceted search for your products. We will use extension `pg_trgm` for searching products with name similar to provided by user.

What we need to implement it?

1) [pg_trgm](<http://www.postgresql.org/docs/current/static/pgtrgm.html>) extension.
{% highlight sql %}
CREATE EXTENSION IF NOT EXISTS pg_trgm;
{% endhighlight %}

2) Table to store all this stuff
{% highlight sql %}
CREATE TABLE IF NOT EXISTS product_names (
  product_name  text    NOT NULL,
  product_type  text    NOT NULL,
  product_id    integer NOT NULL
);
CREATE INDEX product_name_gin_index ON product_names
  USING gin(product_name gin_trgm_ops);
{% endhighlight %}

3) Function that will process data. Based on [this](<http://www.postgresql.org/docs/current/static/plpgsql-trigger.html>) example. It's pretty simple. We change our table curresponding to actions on product`s tables.

{% highlight sql %}
CREATE OR REPLACE FUNCTION collect_product_names() RETURNS TRIGGER AS $body$
BEGIN
  IF (TG_OP = 'INSERT') THEN
      INSERT INTO product_names (product_type, product_id, product_name)
      VALUES (TG_TABLE_NAME::TEXT, NEW.id, NEW.name);

      RETURN NULL;
  ELSIF (TG_OP = 'UPDATE') THEN
      UPDATE product_names
         SET product_name = NEW.name
       WHERE  product_id = OLD.id
         AND product_type = TG_TABLE_NAME::TEXT;

      RETURN NULL;
  ELSIF (TG_OP = 'DELETE') THEN
      DELETE
      FROM   product_names
      WHERE  product_type = TG_TABLE_NAME::TEXT
        AND  product_id = OLD.id;

      RETURN NULL;
  ELSE
      RAISE WARNING '[AUDIT.collect_product_names] - Other action occurred: %, at %',TG_OP,now();
      RETURN NULL;
  END IF;
END;
$body$
LANGUAGE plpgsql
SECURITY DEFINER;
{% endhighlight %}

4) Add our function for product`s table
{% highlight sql %}
DROP TRIGGER IF EXISTS collect_product_names ON mobile_phones;
DROP TRIGGER IF EXISTS collect_product_names ON laptops;

CREATE TRIGGER collect_product_names
  AFTER INSERT OR DELETE OR UPDATE ON mobile_phones
  FOR EACH ROW EXECUTE PROCEDURE collect_product_names();
CREATE TRIGGER collect_product_names
  AFTER INSERT OR DELETE OR UPDATE ON laptops
  FOR EACH ROW EXECUTE PROCEDURE collect_product_names();
{% endhighlight %}

5) Migrate existing data to our new table
{% highlight ruby %}
[MobilePhone, Laptop].each do |klass|
  klass.pluck(:name, :id).each do |(name, id)|
    ProductName.create!(product_name: name,
      product_id: id, product_type: klass.table_name)
  end
end
{% endhighlight %}

All in one

{% highlight ruby %}
class AddProductNamesTrigger < ActiveRecord::Migration
  def up
    execute %Q{
      CREATE EXTENSION IF NOT EXISTS pg_trgm;

      CREATE TABLE IF NOT EXISTS product_names (
          product_name  text    NOT NULL,
          product_type  text    NOT NULL,
          product_id    integer NOT NULL
      );
      CREATE INDEX product_name_gin_index ON product_names USING gin(product_name gin_trgm_ops);

      CREATE OR REPLACE FUNCTION collect_product_names() RETURNS TRIGGER AS $body$
        BEGIN
          IF (TG_OP = 'INSERT') THEN
              INSERT INTO product_names (product_type, product_id, product_name)
              VALUES (TG_TABLE_NAME::TEXT, NEW.id, NEW.name);

              RETURN NULL;
          ELSIF (TG_OP = 'UPDATE') THEN
              UPDATE product_names
                 SET product_name = NEW.name
               WHERE  product_id = OLD.id
                 AND product_type = TG_TABLE_NAME::TEXT;

              RETURN NULL;
          ELSIF (TG_OP = 'DELETE') THEN
              DELETE
              FROM   product_names
              WHERE  product_type = TG_TABLE_NAME::TEXT
                AND  product_id = OLD.id;

              RETURN NULL;
          ELSE
              RAISE WARNING '[AUDIT.collect_product_names] - Other action occurred: %, at %',TG_OP,now();
              RETURN NULL;
          END IF;

        END;
      $body$
      LANGUAGE plpgsql
      SECURITY DEFINER;

      DROP TRIGGER IF EXISTS collect_product_names ON mobile_phones;
      DROP TRIGGER IF EXISTS collect_product_names ON laptops;

      CREATE TRIGGER collect_product_names
        AFTER INSERT OR DELETE OR UPDATE ON mobile_phones
        FOR EACH ROW EXECUTE PROCEDURE collect_product_names();
      CREATE TRIGGER collect_product_names
        AFTER INSERT OR DELETE OR UPDATE ON laptops
        FOR EACH ROW EXECUTE PROCEDURE collect_product_names();
    }
    [MobilePhone, Laptop].each do |klass|
      klass.pluck(:name, :id).each do |(name, id)|
        ProductName.create(product_name: name, product_id: id, product_type: klass.table_name)
      end
    end
  end

  def down
    execute %Q{
      DROP TABLE IF EXISTS product_names;
      DROP TRIGGER IF EXISTS collect_product_names ON mobile_phones;
      DROP TRIGGER IF EXISTS collect_product_names ON laptops;
      DROP FUNCTION IF EXISTS collect_product_names();
    }
  end
end
{% endhighlight %}



Our databases prepared. Now we need to prepare rails app.

{% highlight ruby %}
class ProductName < ActiveRecord::Base
  scope :by_name,    -> (pattern) { where("product_name % ?", pattern) }
  scope :statistics, -> { group(:product_type).count }
end
{% endhighlight %}

How to use it?

{% highlight ruby %}
ProductName.by_name("Apple").statistics
#=> {"laptops"=>1, "mobile_phones"=>3}
{% endhighlight %}

P.S. Now you can see your function with `\df+` on PostgreSQL terminal.