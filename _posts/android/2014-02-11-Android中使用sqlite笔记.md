---
layout: post
---
{% include JB/setup %}


### 1.实现SQLiteHelper,使用SQLite

代码如下，来自android官网。
{% highlight java %}
public class FeedReaderDbHelper extends SQLiteOpenHelper {
// If you change the database schema, you must increment the database version.
public static final int DATABASE_VERSION = 1;
public static final String DATABASE_NAME = "FeedReader.db";

public FeedReaderDbHelper(Context context) {
    super(context, DATABASE_NAME, null, DATABASE_VERSION);
}

/*(必须重写的方法)
该函数是在第一次创建数据库的时候执行，实际上是在第一次得到SQLiteDatabase对象的时候，才会调用这个方法
* 在SQLiteOpenHelper的getReaderDatabase()或者getWritableDatabase()
* 被第一次调用时都会调用该方法
*/
public void onCreate(SQLiteDatabase db) {
    db.execSQL(SQL_CREATE_ENTRIES);
}
//(必须重写的方法)数据库更新的时候会调用该方法
 public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
    // This database is only a cache for online data, so its upgrade policy is
    // to simply to discard the data and start over
    db.execSQL(SQL_DELETE_ENTRIES);
    onCreate(db);
}
//可以不用重写
public void onDowngrade(SQLiteDatabase db, int oldVersion, int newVersion) {
    onUpgrade(db, oldVersion, newVersion);
}
}
{% endhighlight %}
其中onUpgrade是：

当我们已经有数据库存在，想保存原有数据，只想增加一个字段等，只需修改版本号，

SQLiteOpenHelper会自动判断版本号是否一致，假如不一致会自动调用onUpgrade函数

通过重载这个函数，你可以做任何你想做的事，包括增、删字段等等~

public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
db.execSQL("ALTER TABLE user_table ADD age INTEGER DEFAULT 0");
}

### 2.插入数据

代码如下，来自android官网。
{% highlight java %}
DatabaseHelper dbHelper = new DatabaseHelper(getApplicationContext());
SQLiteDatabase db = dbHelper.getWritableDatabase();
ContentValues cv = new ContentValues();
//key是列名，value是要插入的值                
cv.put("name", "value");
//rowId是返回的新插入的行号，如果插入失败返回-1
long rowId = db.insert("user_table", null, cv);
{% endhighlight %}
其中db.insert("user_table", "column_name", cv);第一个参数是要插入的表名。第二个参数是一个列名，当ContentValues对象为null或者size为0时，该列的值为null。

因为每次调用insert都必须执行insert into tableName(column_name)values(value)，当ContentValues的对象为null等，则会执行insert into tableName(column_name)values(null).

若不为空，则执行正常的插入语句

### 3.查数据 
{% highlight java %}
SQLiteDatabase db = mDbHelper.getReadableDatabase();

// Define a projection that specifies which columns from the database
// you will actually use after this query.
//需要查询返回的列名
String[] projection = {
    FeedEntry._ID,
    FeedEntry.COLUMN_NAME_TITLE,
    FeedEntry.COLUMN_NAME_UPDATED,
    ...
    };

// How you want the results sorted in the resulting Cursor
//查询语句排序
String sortOrder =
    FeedEntry.COLUMN_NAME_UPDATED + " DESC";

Cursor c = db.query(
    FeedEntry.TABLE_NAME,  // The table to query
    projection,                               // The columns to return
    selection,                                // The columns for the WHERE clause
    selectionArgs,                            // The values for the WHERE clause
    null,                                     // don't group the rows
    null,                                     // don't filter by row groups
    sortOrder                                 // The sort order
    );
cursor.moveToFirst();
long itemId = cursor.getLong(
    cursor.getColumnIndexOrThrow(FeedEntry._ID)
);
{% endhighlight %}
