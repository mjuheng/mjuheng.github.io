---
layout: post
title: Kryo序列化工具类
categories: [Utils]
description: Kryo序列化工具类
keywords: Kryo, Utils
---

Kryo序列化工具类

# 引入Jar包
```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo-shaded</artifactId>
    <version>4.0.2</version>
</dependency>
```

# 新建com.esotericsoftware.kryo.serializers.CustomSerializer类
继承FieldSerializer，修改setSerializeTransient值为true，序列化所有字段
```java
package com.esotericsoftware.kryo.serializers;
 
import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Output;
 
public class CustomSerializer<T> extends FieldSerializer<T> {
 
    public CustomSerializer(Kryo kryo, Class type) {
        super(kryo, type, (Class[])null);
        setSerializeTransient(true);
    }
 
    public CustomSerializer(Kryo kryo, Class type, Class[] generics) {
        super(kryo, type, generics, kryo.getFieldSerializerConfig().clone());
        setSerializeTransient(true);
    }
 
    protected CustomSerializer(Kryo kryo, Class type, Class[] generics, FieldSerializerConfig config) {
        super(kryo, type, generics, config);
        setSerializeTransient(true);
    }
}
```

# 编写序列化工具类
```java
import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.Serializer;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;
import com.esotericsoftware.kryo.serializers.CustomSerializer;
import com.esotericsoftware.kryo.serializers.FieldSerializer;
import lombok.extern.slf4j.Slf4j;
import org.activiti.bpmn.model.BpmnModel;
import org.activiti.bpmn.model.Process;
import org.activiti.engine.impl.persistence.deploy.ProcessDefinitionCacheEntry;
import org.activiti.engine.repository.ProcessDefinition;
import org.apache.poi.ss.formula.functions.T;
import org.objenesis.strategy.StdInstantiatorStrategy;
import org.springframework.stereotype.Component;
import sun.reflect.ReflectionFactory;
 
import java.io.*;
import java.lang.reflect.Constructor;
import java.util.concurrent.ConcurrentHashMap;
 
/**
 * 序列化工具类
 *
 * @author 36020
 */
@Slf4j
@Component
public class SerializeUtils {
 
    private final ThreadLocal<Kryo> kryoLocal = ThreadLocal.withInitial(() -> {
        // kyro是线程不安全的，需要为每个线程创建一个独立的对象
        Kryox kryo = new Kryox();
        kryo.setInstantiatorStrategy(new Kryo.DefaultInstantiatorStrategy(
                new StdInstantiatorStrategy()));
        kryo.setDefaultSerializer(CustomSerializer.class);
        return kryo;
    });
 
    public byte[] serialize(Object obj) {
        if (obj == null) {
            return null;
        }
 
        Kryo kryo = kryoLocal.get();
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        Output output = new Output(byteArrayOutputStream);
        kryo.writeClassAndObject(output, obj);
        output.close();
        return byteArrayOutputStream.toByteArray();
    }
 
 
    public Object deSerialize(byte[] bytes) {
        if (bytes == null) {
            return null;
        }
 
        Kryo kryo = kryoLocal.get();
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
        Input input = new Input(byteArrayInputStream);
        input.close();
        return kryo.readClassAndObject(input);
    }
 
 
    /**
     * Kyro序列化没有无参构造方法的类时会报错，所以进行改造
     */
    class Kryox extends Kryo {
 
        private final ReflectionFactory REFLECTION_FACTORY = ReflectionFactory.getReflectionFactory();
 
        private final ConcurrentHashMap<Class<?>, Constructor<?>> _constructors = new ConcurrentHashMap<Class<?>, Constructor<?>>();
 
        @Override
        public <T> T newInstance(Class<T> type) {
            try {
                return super.newInstance(type);
            } catch (Exception e) {
                return (T) newInstanceFromReflectionFactory(type);
            }
        }
 
        private Object newInstanceFrom(Constructor<?> constructor) {
            try {
                return constructor.newInstance();
            } catch (final Exception e) {
                throw new RuntimeException(e);
            }
        }
 
        @SuppressWarnings("unchecked")
        public <T> T newInstanceFromReflectionFactory(Class<T> type) {
            Constructor<?> constructor = _constructors.get(type);
            if (constructor == null) {
                constructor = newConstructorForSerialization(type);
                Constructor<?> saved = _constructors.putIfAbsent(type, constructor);
                if (saved != null)
                    constructor = saved;
            }
            return (T) newInstanceFrom(constructor);
        }
 
        private <T> Constructor<?> newConstructorForSerialization(Class<T> type) {
            try {
                Constructor<?> constructor = REFLECTION_FACTORY.newConstructorForSerialization(type,
                        Object.class.getDeclaredConstructor());
                constructor.setAccessible(true);
                return constructor;
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
 
    }
}
```