# 基于Spring Session实现JIM分布式Session
## 前引
在实际项目中，应用程序经常会以集群方式部署线上，一般来说无状态的应用程序是理想的部署方式，一旦应用程序拥有状态（比如Session、线程内缓存、唯一全局ID等），那么会出现状态之间无法共享，对此通常解决方案时：使用第三方系统进行状态的分布式统一管理，比如Redis、Memecache、Zookeeper等。<br>

我们经常会在Web程序使用Session，需要分布式Session。 分布式Session有几种实现方式，这里不加阐述，有兴趣的话，大家可以百度搜索下。这里使用最常用的方式：基于Spring Session实现<br>

网上很多直接使用SpringSession + Redis实现分布式Session，由于SpringSession实现了Redis Session，直接简单集成就可以，但如何集成其他第三方KV缓存就很少有该文章。以下就是基于SpringSession实现第三方KV缓存实现分布式Session。

JIM是第三方KV缓存，也是基于Redis改造的KV缓存
## 代码实现
1. 引用Spring Session的Maven
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session</artifactId>
    <version>1.2.2.RELEASE</version>
</dependency>
```

2. 定义注解EnableJimHttpSession
```java
package com.jd.wl.stockctrl.web.session;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.TYPE})
@Documented
@Import(JimHttpSessionConfiguration.class)
@Configuration
public @interface EnableJimHttpSession {
    int maxInactiveIntervalInSeconds() default 1800;
}

```

3. 继承SpringHttpSessionConfiguration
```java
package com.jd.wl.stockctrl.web.session;

import org.springframework.cache.Cache;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportAware;
import org.springframework.core.annotation.AnnotationAttributes;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.session.config.annotation.web.http.SpringHttpSessionConfiguration;

import java.util.Map;

/**
 * 继承SpringHttpSessionConfiguration
 * @author sunchangtan
 * @date 2019/1/23 20:51
 */

@Configuration
@EnableScheduling
public class JimHttpSessionConfiguration extends SpringHttpSessionConfiguration implements ImportAware {
    private Integer maxInactiveIntervalInSeconds = 1800;

    @Override
    public void setImportMetadata(AnnotationMetadata importMetadata) {
        Map<String, Object> enableAttrMap = importMetadata.getAnnotationAttributes(EnableJimHttpSession.class.getName());
        AnnotationAttributes enableAttrs = AnnotationAttributes.fromMap(enableAttrMap);
        this.maxInactiveIntervalInSeconds = enableAttrs.getNumber("maxInactiveIntervalInSeconds");
    }

    @Bean
    public JimOperationsSessionRepository jimSessionRepository(Cache cache) {
        JimOperationsSessionRepository repository = new JimOperationsSessionRepository(cache);
        repository.setMaxInactiveIntervalInSeconds(this.maxInactiveIntervalInSeconds);
        return repository;
    }
}

```

4. 核心逻辑实现FindByIndexNameSessionRepository接口，定义KV缓存的CRUD操作
```java
package com.jd.wl.stockctrl.web.session;

import com.google.common.collect.Maps;
import com.jd.wl.stockctrl.service.cache.jimdb.CacheCluster;
import com.jd.wl.stockctrl.service.cache.jimdb.JimdbCache;
import com.jd.wl.stockctrl.service.cache.jimdb.SingleJimdbCache;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.Cache;
import org.springframework.session.ExpiringSession;
import org.springframework.session.FindByIndexNameSessionRepository;
import org.springframework.session.MapSession;
import org.springframework.util.Assert;
import sun.reflect.generics.reflectiveObjects.NotImplementedException;

import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 基于JIM的session操作
 * @author sunchangtan
 * @date 2019/1/23 21:08
 */
@Slf4j
public class JimOperationsSessionRepository implements FindByIndexNameSessionRepository<JimOperationsSessionRepository.JimSession> {
    private static final String DEFAULT_SPRING_SESSION_REDIS_PREFIX = "spring:session:";

    private static final String CREATION_TIME_ATTR = "creationTime";

    private static final String MAX_INACTIVE_ATTR = "maxInactiveInterval";

    private static final String LAST_ACCESSED_ATTR = "lastAccessedTime";

    private static final String SESSION_ATTR_PREFIX = "sessionAttr:";

    private Integer defaultMaxInactiveInterval;

    private final Cache cache;

    public JimOperationsSessionRepository(Cache cache) {
        this.cache = cache;
    }

    void setMaxInactiveIntervalInSeconds(Integer maxInactiveIntervalInSeconds) {
        this.defaultMaxInactiveInterval = maxInactiveIntervalInSeconds;
    }

    @Override
    public Map<String, JimSession> findByIndexNameAndIndexValue(String indexName, String indexValue) {
        return null;
    }

    @Override
    public JimSession createSession() {
        JimOperationsSessionRepository.JimSession jimSession = new JimOperationsSessionRepository.JimSession();
        if (this.defaultMaxInactiveInterval != null) {
            jimSession.setMaxInactiveIntervalInSeconds(this.defaultMaxInactiveInterval);
        }
        return jimSession;
    }

    /**
     * 获取JIMDB集群对象， 可以自定义
     */
    private CacheCluster getCacheCluster() {
        if (cache instanceof SingleJimdbCache) {
            return ((SingleJimdbCache) cache).getNativeCache();
        } else if (cache instanceof JimdbCache) {
            return ((JimdbCache) cache).getNativeCache();
        } else {
            throw new NotImplementedException();
        }
    }

    /**
     * 保存SESSION对象
     * @param session
     */
    @Override
    public void save(JimSession session) {
        session.saveDelta();
    }

    /**
     * 获取SESSION对象
     * @param sessionId
     * @return
     */
    @Override
    public JimSession getSession(String sessionId) {
        return getSession(sessionId, false);
    }

    /**
     * 删除对象
     * @param sessionId
     */
    @Override
    public void delete(String sessionId) {
        JimOperationsSessionRepository.JimSession session = getSession(sessionId, true);
        if (session == null) {
            return;
        }
        String expireKey = getSessionKey(sessionId);
        this.getCacheCluster().delObject(expireKey);

        session.setMaxInactiveIntervalInSeconds(0);
        save(session);
    }

    /**
    * 保存Session数据
    */
    private void save(String sessionId, Map<String, Object> sessionData) {
        final String key = getSessionKey(sessionId);

        final CacheCluster cacheCluster = this.getCacheCluster();
        sessionData.forEach((mapKey, mapValue) -> cacheCluster.setHObject(key, mapKey, ""+mapValue));
    }

    /**
    * 获取Session数据
    */
    @SuppressWarnings("unchecked")
    private JimSession getSession(String sessionId, boolean allowExpired) {
        String key = getSessionKey(sessionId);

        try {
            Map<String, String> obj = this.getCacheCluster().getHObject(key);
            if (obj == null || obj.isEmpty()) {
                return null;
            }

            Map<Object, Object> entries = Maps.newHashMapWithExpectedSize(obj.size());
            obj.forEach(entries::put);

            MapSession loaded = loadSession(sessionId, entries);
            if (!allowExpired && loaded.isExpired()) {
                return null;
            }

            JimSession result = new JimSession(loaded);
            result.originalLastAccessTime = loaded.getLastAccessedTime();
            return result;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }

        return null;
    }

    private MapSession loadSession(String id, Map<Object, Object> entries) {
        MapSession loaded = new MapSession(id);
        for (Map.Entry<Object, Object> entry : entries.entrySet()) {
            String key = (String) entry.getKey();
            if (CREATION_TIME_ATTR.equals(key)) {
                loaded.setCreationTime(Long.valueOf(entry.getValue().toString()));
            } else if (MAX_INACTIVE_ATTR.equals(key)) {
                loaded.setMaxInactiveIntervalInSeconds(Integer.valueOf(entry.getValue().toString()));
            } else if (LAST_ACCESSED_ATTR.equals(key)) {
                loaded.setLastAccessedTime(Long.valueOf(entry.getValue().toString()));
            } else if (key.startsWith(SESSION_ATTR_PREFIX)) {
                loaded.setAttribute(key.substring(SESSION_ATTR_PREFIX.length()),
                        entry.getValue());
            }
        }
        return loaded;
    }

    private String getSessionAttrNameKey(String attributeName) {
        return SESSION_ATTR_PREFIX + attributeName;
    }

    private String getSessionKey(String sessionId) {
        return DEFAULT_SPRING_SESSION_REDIS_PREFIX + "sessions:" + sessionId;
    }

    /**
     * 封装MappSession操作session对象
     */
    final class JimSession implements ExpiringSession {
        private final MapSession cached;
        private Map<String, Object> delta = new HashMap<>();

        long originalLastAccessTime;

        JimSession() {
            this(new MapSession());
            this.delta.put(CREATION_TIME_ATTR, getCreationTime());
            this.delta.put(MAX_INACTIVE_ATTR, getMaxInactiveIntervalInSeconds());
            this.delta.put(LAST_ACCESSED_ATTR, getLastAccessedTime());
        }

        JimSession(MapSession cached) {
            Assert.notNull(cached, "MapSession cannot be null");
            this.cached = cached;
        }

        @Override
        public void setLastAccessedTime(long lastAccessedTime) {
            this.cached.setLastAccessedTime(lastAccessedTime);
            this.delta.put(LAST_ACCESSED_ATTR, getLastAccessedTime());
            saveDelta();
        }

        @Override
        public boolean isExpired() {
            return this.cached.isExpired();
        }

        @Override
        public long getCreationTime() {
            return this.cached.getCreationTime();
        }

        @Override
        public String getId() {
            return this.cached.getId();
        }

        @Override
        public long getLastAccessedTime() {
            return this.cached.getLastAccessedTime();
        }

        @Override
        public void setMaxInactiveIntervalInSeconds(int interval) {
            this.cached.setMaxInactiveIntervalInSeconds(interval);
            this.delta.put(MAX_INACTIVE_ATTR, getMaxInactiveIntervalInSeconds());
            saveDelta();
        }

        @Override
        public int getMaxInactiveIntervalInSeconds() {
            return this.cached.getMaxInactiveIntervalInSeconds();
        }

        /**
        * 获取Session中的属性值
        */
        @SuppressWarnings("unchecked")
        @Override
        public Object getAttribute(String attributeName) {
            return this.cached.getAttribute(attributeName);
        }

        @Override
        public Set<String> getAttributeNames() {
            return this.cached.getAttributeNames();
        }

        /**
        * 设置Session中的属性值
        */
        @Override
        public void setAttribute(String attributeName, Object attributeValue) {
            this.cached.setAttribute(attributeName, attributeValue);
            this.delta.put(getSessionAttrNameKey(attributeName), attributeValue);
            saveDelta();
        }

        /**
        * 删除Session中的属性值
        */
        @Override
        public void removeAttribute(String attributeName) {
            this.cached.removeAttribute(attributeName);
            this.delta.put(getSessionAttrNameKey(attributeName), null);
            saveDelta();
        }

        /**
         * 保存增量的session属性
         */
        private void saveDelta() {
            if (this.delta.isEmpty()) {
                return;
            }
            String sessionId = getId();
            save(sessionId, this.delta);

            this.delta.clear();
        }

    }

}
```

5. SpringBoot的主程序入口中注解@EnableJimHttpSession
```java
@EnableJimHttpSession
@SpringBootApplication
public class Application extends SpringBootServletInitializer {
        public static void main(String[] args) throws Exception {
        SpringApplicationBuilder builder = new SpringApplicationBuilder(Application.class);
        builder.bannerMode(Banner.Mode.OFF);
        builder.run(args);
    }
}
```