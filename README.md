# ServletContainerInitializer-example

This project is designed to demonstrate a missing feature of Liferay Portal (and OSGi Servlet Containers in general). When a dependency jar with a **`META-INF/services/javax.servlet.ServletContainerInitializer`** is included inside a war's **`WEB-INF/lib`** it will be detected and run. However, when that dependency is extracted from **`WEB-INF/lib`** and included as an OSGi module instead, the `ServletContainerInitializer` will not be run. If possible, the dependencies of the bundle should be searched for any **`META-INF/services/javax.servlet.ServletContainerInitializer`** files which should then be run at the appropriate time.

## Steps to reproduce:

1. Start Liferay Portal 7.0+.
2. Declare the location of Liferay Portal's home directory:

        LIFERAY_HOME=/path/to/liferay-portal-7.0

3. Remove any previous versions of these artifacts:

        rm $LIFERAY_HOME/osgi/war/jsp-war-portlet*.war $LIFERAY_HOME/osgi/modules/ServletContainerInitializer-dependency*.jar

4. Build the project as a fat war:

        mvn clean package

5. Copy the war to Liferay's deploy folder:

        cp jsp-war-portlet/target/jsp-war-portlet-1.0.war $LIFERAY_HOME/deploy/jsp-war-portlet.war

6. Examine the Liferay logs to confirm that the following logs appear indicating that `ExampleServletContainerInitializer.onStartup()` was executed:

    ```
    21:16:03,202 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:37]             -------------------------------------------------------------------------------------
    21:16:03,203 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:38]             The following classes were annotated with com.liferay.example.servlet.container.initializer.dependency.ExampleAnnotation:
    21:16:03,203 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:42]             com.liferay.blade.samples.portlet.ExampleAnnotatedClass
    21:16:03,203 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:46]             -------------------------------------------------------------------------------------
    ```

7. Remove any previous versions of these artifacts:

        rm $LIFERAY_HOME/osgi/war/jsp-war-portlet*.war $LIFERAY_HOME/osgi/modules/ServletContainerInitializer-dependency*.jar

8. Build the project as a thin war:

        mvn clean package -P thin-war

9. Copy the **`ServletContainerInitializer-dependency.jar`** to Liferay's **`osgi/modules`** folder:

        cp ServletContainerInitializer-dependency/target/ServletContainerInitializer-dependency-1.0.jar $LIFERAY_HOME/osgi/modules/.

10. Wait until that module has started.
11. Copy the war to Liferay's deploy folder:

        cp jsp-war-portlet/target/jsp-war-portlet-1.0.war $LIFERAY_HOME/deploy/jsp-war-portlet.war

If the feature is complete the logs will appear the same as when the dependency was packaged within the war (indicating that `ExampleServletContainerInitializer.onStartup()` executed):

```
21:16:03,202 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:37]             -------------------------------------------------------------------------------------
21:16:03,203 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:38]             The following classes were annotated with com.liferay.example.servlet.container.initializer.dependency.ExampleAnnotation:
21:16:03,203 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:42]             com.liferay.blade.samples.portlet.ExampleAnnotatedClass
21:16:03,203 FATAL [fileinstall-/home/kylestiemann/Portals/liferay.com/7.0/osgi/war][ExampleServletContainerInitializer:46]             -------------------------------------------------------------------------------------
```

If the feature is not complete `ExampleServletContainerInitializer.onStartup()` will not be executed and no logs will appear.

## Notes

In order to find the appropriate `ServletContainerInitializer`s, the dependencies of the bundle must be found using the OSGi API. It may or may not be necessary to recursively find the dependencies. The following code would need to appear in [`WabBundleProcessor.init(Dictionary<String, Object> properties)`](https://github.com/liferay/liferay-portal/blob/7.0.4-ga5/modules/apps/static/portal-osgi-web/portal-osgi-web-wab-extender/src/main/java/com/liferay/portal/osgi/web/wab/extender/internal/WabBundleProcessor.java#L123-L234):

```
BundleWiring bundleWiring = bundle.adapt(BundleWiring.class);
List<BundleWire> bundleWires = bundleWiring.getRequiredWires(BundleRevision.PACKAGE_NAMESPACE);

for (BundleWire bundleWire : bundleWires) {

	bundle = bundleWire.getProvider().getBundle();

	if (bundle.getBundleId() != 0) {

		// See here for more details:
		// https://github.com/liferay/liferay-portal/blob/7.0.4-ga5/modules/apps/static/portal-osgi-web/portal-osgi-web-wab-extender/src/main/java/com/liferay/portal/osgi/web/wab/extender/internal/WabBundleProcessor.java#L590-L634
		/*wabBundleProcessor.*/initServletContainerInitializers(bundle, servletContext);

		// Consider searching the dependencies of the current bundle here to ensure that
		// transitive dependencies are searched.
	}
}
```

### Copy/pastable Commands to Test Thick War: 

```
([ -n "$LIFERAY_HOME" ] && ls $LIFERAY_HOME > /dev/null) &&
(rm $LIFERAY_HOME/osgi/war/jsp-war-portlet*.war $LIFERAY_HOME/osgi/modules/ServletContainerInitializer-dependency*.jar; \
mvn clean package &&
cp jsp-war-portlet/target/jsp-war-portlet-1.0.war $LIFERAY_HOME/deploy/jsp-war-portlet.war) ||
echo "Error: LIFERAY_HOME not set or not valid directory."
```

### Copy/pastable Commands to Test Thin War: 

```
([ -n "$LIFERAY_HOME" ] && ls $LIFERAY_HOME > /dev/null) &&
(rm $LIFERAY_HOME/osgi/war/jsp-war-portlet*.war $LIFERAY_HOME/osgi/modules/ServletContainerInitializer-dependency*.jar; \
mvn clean package -P thin-war &&
cp ServletContainerInitializer-dependency/target/ServletContainerInitializer-dependency-1.0.jar $LIFERAY_HOME/osgi/modules/. &&
sleep 15 &&
cp jsp-war-portlet/target/jsp-war-portlet-1.0.war $LIFERAY_HOME/deploy/jsp-war-portlet.war) ||
echo "Error: LIFERAY_HOME not set or not valid directory."
```