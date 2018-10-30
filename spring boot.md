###一.SpringBoot入门

```java
package com.csnt.scdp.bizmodules.cache.cs.mp;

import com.csnt.scdp.bizmodules.helper.cs.VScdpOrgHelper;
import com.csnt.scdp.framework.attributes.CommonAttribute;
import com.csnt.scdp.framework.core.cache.BaseCache;
import com.csnt.scdp.framework.core.logtracer.LogTracerFactory;
import com.csnt.scdp.framework.core.logtracer.intf.ILogTracer;
import com.csnt.scdp.framework.dto.CacheType;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

/**
 * Description:  组织机构缓存设置 公司-管理处-收费站
 * Copyright: © 2016 CSNT. All rights reserved.
 * Company:CSNT
 *
 * @author wangling
 * @version 1.0
 * @timestamp 2016-08-28 15:39:29
 */
@Scope("singleton")
@Component("CACHE_V_SCDP_ORG_DATA")
@CacheType(value = CommonAttribute.CACHE_EHCACHE_IMPL, cacheName = "CACHE_V_SCDP_ORGD")
public class VScdpOrgCache extends BaseCache {
    private static ILogTracer tracer = LogTracerFactory.getInstance(VScdpOrgCache.class);
    public Object getFreshValue(Object cacheKey) {
        try {
            return VScdpOrgHelper.loadVScdpOrg();
        } catch (Exception e) {
            tracer.error(e);
        }
        return null;
    }
}
```

