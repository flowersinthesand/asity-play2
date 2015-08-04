# Asity bridge for Play framework 2

An [Asity](http://asity.cettia.io) bridge for [Play framework 2](http://www.playframework.org/) to allow run an Asity application on Play framework 2.

# Recipe

Add the following dependency to your `build.sbt` or include it on your classpath manually.

```scala
libraryDependencies ++= Seq(
  "io.cettia.asity" % "asity-bridge-play2" % "1.0.0-Beta1"
)
```

Then, write entry point for HTTP exchange and WebSocket extending `Controller`. A helper class will be introduced solving the above notes.

```java
public class Bootstrap extends Controller {
    // Your application
    static Action<ServerHttpExchange> httpAction = http -> {};
    static Action<ServerWebSocket> websocketAction = ws -> {};

    @BodyParser.Of(BodyParser.Raw.class)
    public static Promise<Result> http() {
        PlayServerHttpExchange http = new PlayServerHttpExchange(request(), response());
        httpAction.on(http);
        return http.result();
    }
    
    public static WebSocket<String> websocket() {
        final Http.Request request = request();
        return new WebSocket<String>() {
            @Override
            public void onReady(WebSocket.In<String> in, WebSocket.Out<String> out) {
                websocketAction.on(new PlayServerWebSocket(request, in, out));
            }
        };
    }
}

```

Play doesn't allow to share URI between HTTP and WebSocket entry points. Instead of `routes`, write `Global.scala` in the default package and override `onRouteRequest`. It's not easy to do that in Java, if any. Note that this uses internal API that has broken even in patch release. I've confirmed the following code works in `2.2.2` and `2.3.2`.

```scala
import io.cettia.example.platform.play2.{Bootstrap => T}

import play.api.GlobalSettings
import play.api.mvc._
import play.core.j._

object Global extends GlobalSettings {
  override def onRouteRequest(req: RequestHeader): Option[Handler] = {
    if (req.path == "/cettia") {
      if (req.method == "GET" && req.headers.get("Upgrade").exists(_.equalsIgnoreCase("websocket"))) {
        Some(JavaWebSocket.ofString(T.websocket))
      } else {
        Some(new JavaAction {
          val annotations = new JavaActionAnnotations(classOf[T], classOf[T].getMethod("http"))
          val parser = annotations.parser
          def invocation = T.http
        })
      }
    } else {
      super.onRouteRequest(req)
    }
  }
}
```

## Roadmap

* Support Play 2.4.
* Rewrite in Scala [#1](https://github.com/flowersinthesand/asity-play2/issues/1).
* Add tests extending ServerHttpExchangeTestBase and ServerWebSocketTestBase.
* Add examples.
* Provide a helper class.
