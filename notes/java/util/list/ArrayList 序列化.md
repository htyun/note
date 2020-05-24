# ArrayList 序列化

> 我们先来看看 ArrayList 部分源码

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;
    // non-private to simplify nested class access
    // 非私有以简化嵌套类访问
    transient Object[] elementData; 
    private int size;
}
```

我们可以看到 elementData 是 `transient`，了解序列化的同志们应该都知道，
默认序列化，为了不序列化某个变量，会在成员变量前面加 `transient`。

我们现在先写一个列子：

```java

public class ArrayListSerializable {

    public static void main(String[] args) {
        List<Student> studentList = new ArrayList<>();
        studentList.add(new Student("Ken", 23));
        studentList.add(new Student("dsxsb", 24));
        studentList.add(new Student("Rose", 25));

        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("arrayListSerializable"))){
            oos.writeObject(studentList);
        } catch (IOException e) {
            e.printStackTrace();
        }

        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("arrayListSerializable"))){
            List<Student> objList = (List<Student>) ois.readObject();
            System.out.println(objList);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

class Student implements Serializable {
    private static final long serialVersionUID = -5675902862216268035L;
    private String name;
    private Integer age;

    public Student(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

// sout [Student{name='Ken', age=23}, Student{name='dsxsb', age=24}, Student{name='Rose', age=25}]
```

ArrayList 底层是通过，elementData 又是被标记的变量，为什么还会被序列化保存下来？

> 我们先介绍一下 ObjectInputStream

```java
    public final void writeObject(Object obj) throws IOException {
        if (enableOverride) {
            writeObjectOverride(obj);
            return;
        }
        try {
            writeObject0(obj, false);
        } catch (IOException ex) {
            if (depth == 0) {
                writeFatalException(ex);
            }
            throw ex;
        }
    }

    private void writeObject0(Object obj, boolean unshared)
        throws IOException {
        ...

            // remaining cases
            if (obj instanceof String) {
                writeString((String) obj, unshared);
            } else if (cl.isArray()) {
                writeArray(obj, desc, unshared);
            } else if (obj instanceof Enum) {
                writeEnum((Enum<?>) obj, desc, unshared);
            } else if (obj instanceof Serializable) {
                // 实现 Serializable 接口会走这里
                writeOrdinaryObject(obj, desc, unshared);
            } else {
                if (extendedDebugInfo) {
                    throw new NotSerializableException(
                        cl.getName() + "\n" + debugInfoStack.toString());
                } else {
                    throw new NotSerializableException(cl.getName());
                }
            }
        
        ...
    }

    private void writeOrdinaryObject(Object obj,
                                     ObjectStreamClass desc,
                                     boolean unshared)
        throws IOException
    {
       ...
            if (desc.isExternalizable() && !desc.isProxy()) {
                writeExternalData((Externalizable) obj);
            } else {
                writeSerialData(obj, desc);
            }
       ...
    }

    private void writeSerialData(Object obj, ObjectStreamClass desc)
        throws IOException
    {
        ...
                    curContext = new SerialCallbackContext(obj, slotDesc);
                    bout.setBlockDataMode(true);
                    // 注意这个方法
                    slotDesc.invokeWriteObject(obj, this);
                    bout.setBlockDataMode(false);
                    bout.writeByte(TC_ENDBLOCKDATA);
       ...
    }

    void invokeWriteObject(Object obj, ObjectOutputStream out)
        throws IOException, UnsupportedOperationException
    {
        requireInitialized();
        if (writeObjectMethod != null) {
            try {
                // 通过反射来调用方法
                writeObjectMethod.invoke(obj, new Object[]{ out });
            } catch (InvocationTargetException ex) {
                Throwable th = ex.getTargetException();
                if (th instanceof IOException) {
                    throw (IOException) th;
                } else {
                    throwMiscException(th);
                }
            } catch (IllegalAccessException ex) {
                // should not occur, as access checks have been suppressed
                throw new InternalError(ex);
            }
        } else {
            throw new UnsupportedOperationException();
        }
    }
```
上面方法调用是：
`writeObject() --> writeObject0() --> writeOrdinaryObject() --> writeSerialData() --> invokeWriteObject()`

`invokeWriteObject()` 方法注释 `Invokes the writeObject method of the represented serializable class.`
翻译就是`调用所表示的可序列化类的 writeObject 方法。`
所以就是他会去通过反射调用开发者自定义的 writeObject()。这就是突破口，是否 ArrayList 有自定义的 writeObject()，我们找找看。

找啊找，哎呀，果然在 ArrayList 源码中找到了 writeObject() 方法：
```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```

看到上面循环里的 `s.writeObject(elementData[i]);` 没，通过自定义 writeObject() 对数组元素进行序列化。
从侧面也能看出，如果想要序列化 ArrayList ，它的元素对象也是需要可序列化的，也是可以自己重写 writeObject();

# 总结

1. 到这里我以从 ArrayList 序列化了解到了，如果一个类想被序列化，需要实现Serializable接口。
2. 在对象中实现自定义方法 writeObject() 和 readObject() 方法可以实现自定义序列化。