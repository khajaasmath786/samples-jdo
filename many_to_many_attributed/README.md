# Many-to-Many Attributed

The prerequisite is that you need the DataNucleus Maven plugin installed. You can download this plugin from the DataNucleus downloads area.

* Set the database configuration in "src/main/resources/META-INF/persistence.xml"
* Make sure you have the JDBC driver jar specified in the "pom.xml" file
* Run the command: "mvn clean compile". This builds everything, and enhances the classes
* Run the command: "mvn datanucleus:schema-create". This creates the schema
* Run some persistence code. "mvn clean exec:java"
* Run the command: "mvn datanucleus:schema-delete". This deletes the schema


# Guide

DataNucleus provides support for standard JDO M-N relations where we have a relation between, for example, _Customer_ and _Supplier_, 
where a _Customer_ has many _Suppliers_ and a _Supplier_ has many _Customers_. 
A slight modification on this is where you have the relation carrying some additional attributes of the relation. Let's take some classes

```
public class Customer
{
    private long id; // PK
    private String name;
    private Set supplierRelations = new HashSet();

    ...
}

public class Supplier
{
    private long id; // PK
    private String name;
    private Set customerRelations = new HashSet();

    ...
}
```

Now we obviously can't define an "attributed relation" using Java and just these classes so we invent an intermediate "associative" class, that will also contain the attributes.

```
public class BusinessRelation
{
    private Customer customer; // PK
    private Supplier supplier; // PK
    private String relationLevel;
    private String meetingLocation;

    public BusinessRelation(Customer cust, Supplier supp, String level, String meeting)
    {
        this.customer = cust;
        this.supplier = supp;
        this.relationLevel = level;
        this.meetingLocation = meeting;
    }
    ...
}
```

So we define the metadata like this

```
<jdo>
    <package name="mydomain.business">
        <class name="Customer" detachable="true" table="CUSTOMER">
            <field name="id" primary-key="true" value-strategy="increment" column="ID"/>
            <field name="name" column="NAME"/>
            <field name="supplierRelations" persistence-modifier="persistent" mapped-by="customer">
                <collection element-type="BusinessRelation"/>
            </field>
        </class>

        <class name="Supplier" detachable="true" table="SUPPLIER">
            <field name="id" primary-key="true" value-strategy="increment" column="ID"/>
            <field name="name" column="NAME"/>
            <field name="customerRelations" persistence-modifier="persistent" mapped-by="supplier">
                <collection element-type="BusinessRelation"/>
            </field>
        </class>

        <class name="BusinessRelation" type="application" detachable="true"
             objectid-class="BusinessRelation$PK" table="BUSINESSRELATION">
            <field name="customer" primary-key="true" column="CUSTOMER_ID"/>
            <field name="supplier" primary-key="true" column="SUPPLIER_ID"/>
            <field name="relationLevel" column="RELATION_LEVEL"/>
            <field name="meetingLocation" column="MEETING_LOCATION"/>
        </class>
    </package>
</jdo>
```

So we've used a 1-N "CompoundIdentity" relation between _Customer_ and _BusinessRelation_, and similarly between _Supplier_ and _BusinessRelation_ 
meaning that _BusinessRelation_ has a composite PK define like this

```
public class BusinessRelation
{
    ...

    public static class PK implements Serializable
    {
        public LongIdentity customer; // Use same name as BusinessRelation field
        public LongIdentity supplier; // Use same name as BusinessRelation field

        public PK()
        {
        }

        public PK(String s)
        {
            StringTokenizer st = new StringTokenizer(s, "::");
            this.customer = new LongIdentity(Customer.class, st.nextToken());
            this.supplier = new LongIdentity(Supplier.class, st.nextToken());
        }

        public String toString()
        {
            return (customer.toString() + "::" + supplier.toString());
        }

        public int hashCode()
        {
            return customer.hashCode() ^ supplier.hashCode();
        }

        public boolean equals(Object other)
        {
            if (other != null && (other instanceof PK))
            {
                PK otherPK = (PK)other;
                return this.customer.equals(otherPK.customer) && this.supplier.equals(otherPK.supplier);
            }
            return false;
        }
    }
}
```

This arrangement will result in the following schema

![schema_image](docs/many_to_many_attributed.jpg)


So all we need to do now is persist some objects using these classes

```
PersistenceManagerFactory pmf = JDOHelper.getPersistenceManagerFactory("datanucleus.properties");
PersistenceManager pm = pmf.getPersistenceManager();
Transaction tx = pm.currentTransaction();
Object holderId = null;
try
{
    tx1.begin();

    Customer cust1 = new Customer("Web design Inc");
    Supplier supp1 = new Supplier("DataNucleus Corporation");
    pm.makePersistent(cust1);
    pm.makePersistent(supp1);

    BusinessRelation rel_1_1 = new BusinessRelation(cust1, supp1, "Very Friendly", "Hilton Hotel, London");
    cust1.addRelation(rel_1_1);
    supp1.addRelation(rel_1_1);
    pm.makePersistent(rel_1_1);

    tx.commit();
}
finally
{
    if (tx1.isActive())
    {
        tx1.rollback();
    }
    pm1.close();
}
```

This will now have persisted an entry in table "CUSTOMER", an entry in table "SUPPLIER", and an entry in table "BUSINESSRELATION". 
We can now utilise the _BusinessRelation_ objects to update the attributes of the M-N relation as we wish.


