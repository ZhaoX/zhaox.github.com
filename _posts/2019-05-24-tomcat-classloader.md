---
layout: post
title: "Tomcat的类加载机制"
description: "Tomcat ClassLoader"
category: Java
tags: [Java, Web, Tomcat]
---
{% include JB/setup %}

Tomcat是Servlet容器，首先看Servlet标准对类加载的要求

### Servlet Specification, version 2.4, section 9.7.2: Web Application Class Loader

> The class loader that a container uses to load a servlet in a WAR must allow the
> developer to load any resources contained in library JARs within the WAR
> following normal J2SE semantics using getResource. As described in the J2EE
> license agreement, servlet containers that are not part of a J2EE product should not
> allow the application to override J2SE platform classes, such as those in the java.*
> and javax.* namespaces, that J2SE does not allow to be modified. Also, servlet
> containers that are part of a J2EE product should not allow the application to
> override J2SE or J2EE platform classes, such as those in java.* and javax.*
> namespaces, that either J2SE or J2EE do not allow to be modified. The container
> should not allow applications to override or access the container’s implementation
> classes. It is recommended also that the application class loader be implemented so
> that classes and resources packaged within the WAR are loaded in preference to
> classes and resources residing in container-wide library JARs.

### Class Loader Hierarchy

      Bootstrap
          |
       System
          |
       Common
       /     \
  Webapp1   Webapp2 ...


      Bootstrap
        |
      System
        |
      Common
       /  \
  Server  Shared
           /  \
     Webapp1  Webapp2 ...

### Talk is cheap, Show me the code

``` java
package org.apache.catalina.loader;
public abstract class WebappClassLoaderBase extends URLClassLoader
        implements Lifecycle, InstrumentableClassLoader, WebappProperties {
    /**
     * Load the class with the specified name, searching using the following
     * algorithm until it finds and returns the class.  If the class cannot
     * be found, returns <code>ClassNotFoundException</code>.
     * <ul>
     * <li>Call <code>findLoadedClass(String)</code> to check if the
     *     class has already been loaded.  If it has, the same
     *     <code>Class</code> object is returned.</li>
     * <li>If the <code>delegate</code> property is set to <code>true</code>,
     *     call the <code>loadClass()</code> method of the parent class
     *     loader, if any.</li>
     * <li>Call <code>findClass()</code> to find this class in our locally
     *     defined repositories.</li>
     * <li>Call the <code>loadClass()</code> method of our parent
     *     class loader, if any.</li>
     * </ul>
     * If the class was found using the above steps, and the
     * <code>resolve</code> flag is <code>true</code>, this method will then
     * call <code>resolveClass(Class)</code> on the resulting Class object.
     *
     * @param name The binary name of the class to be loaded
     * @param resolve If <code>true</code> then resolve the class
     *
     * @exception ClassNotFoundException if the class was not found
     */
    @Override
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

        synchronized (getClassLoadingLock(name)) {
            if (log.isDebugEnabled())
                log.debug("loadClass(" + name + ", " + resolve + ")");
            Class<?> clazz = null;

            // Log access to stopped class loader
            checkStateForClassLoading(name);

            // (0) Check our previously loaded local class cache
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }

            // (0.1) Check our previously loaded class cache
            clazz = findLoadedClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }

            // (0.2) Try loading the class with the system class loader, to prevent
            //       the webapp from overriding Java SE classes. This implements
            //       SRV.10.7.2
            String resourceName = binaryNameToPath(name, false);

            ClassLoader javaseLoader = getJavaseClassLoader();
            boolean tryLoadingFromJavaseLoader;
            try {
                // Use getResource as it won't trigger an expensive
                // ClassNotFoundException if the resource is not available from
                // the Java SE class loader. However (see
                // https://bz.apache.org/bugzilla/show_bug.cgi?id=58125 for
                // details) when running under a security manager in rare cases
                // this call may trigger a ClassCircularityError.
                tryLoadingFromJavaseLoader = (javaseLoader.getResource(resourceName) != null);
            } catch (ClassCircularityError cce) {
                // The getResource() trick won't work for this class. We have to
                // try loading it directly and accept that we might get a
                // ClassNotFoundException.
                tryLoadingFromJavaseLoader = true;
            }

            if (tryLoadingFromJavaseLoader) {
                try {
                    clazz = javaseLoader.loadClass(name);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }

            // (0.5) Permission to access this class when using a SecurityManager
            if (securityManager != null) {
                int i = name.lastIndexOf('.');
                if (i >= 0) {
                    try {
                        securityManager.checkPackageAccess(name.substring(0,i));
                    } catch (SecurityException se) {
                        String error = "Security Violation, attempt to use " +
                            "Restricted Class: " + name;
                        log.info(error, se);
                        throw new ClassNotFoundException(error, se);
                    }
                }
            }

            boolean delegateLoad = delegate || filter(name);

            // (1) Delegate to our parent if requested
            if (delegateLoad) {
                if (log.isDebugEnabled())
                    log.debug("  Delegating to parent classloader1 " + parent);
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (log.isDebugEnabled())
                            log.debug("  Loading class from parent");
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }

            // (2) Search local repositories
            if (log.isDebugEnabled())
                log.debug("  Searching local repositories");
            try {
                clazz = findClass(name);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from local repository");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }

            // (3) Delegate to parent unconditionally
            if (!delegateLoad) {
                if (log.isDebugEnabled())
                    log.debug("  Delegating to parent classloader at end: " + parent);
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (log.isDebugEnabled())
                            log.debug("  Loading class from parent");
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }
        }

        throw new ClassNotFoundException(name);
    }

}
```

### Reference
http://tomcat.apache.org/tomcat-9.0-doc/class-loader-howto.html
https://download.oracle.com/otndocs/jcp/servlet-2.4-fr-spec-oth-JSpec/
https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-5.html