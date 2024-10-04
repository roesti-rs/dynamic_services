# Dynamic Services

A dynamic service injection framework that provides an Injection style
programming model implemented as macros.

Note that this package is still in experimental stage.

## Injecting Service Consumers

Service consumers are activated and injected with the service as the service
arrives, meaning that the consumer won't exist if the service(s) that it needs
are not there.

Once the service appears, the consumer will be created and injected with the service.

### Example consumer

A very simple consumer using an example TidalService.

```
// Import your service API so you can use it once the instance appears
use crate::tidal_service::TidalService;
use dynamic_services::DynamicServices;
use dynamic_services::{activator, deactivator, dynamic_services};

// Define your consumer struct with at least these derives
#[derive(DynamicServices, Debug, Default)]
pub struct MyConsumer {
  #[inject]
  tidal: Option<ServiceReference<TidalService>>,
}
```
The field annotated with `#[inject]` will be injected by the framework. Right
now only the service reference is injected this way, although a reference to
the actual service is passed to the activator. The `#[inject]` annotation does
not need to be separately imported as it is handled by the `DynamicServices`
derive macro.

Provide an impl for your MyConsumer with the `#[dynamic_services]` annotation.
For now the annotation needs to provide the path to the current impl file,
e.g. if you would declare `use mypkg::consumer::MyConsumer` then `path` must
be `mypkg::consumer`.

```
#[dynamic_services(path=mypkg::consumer)]
impl MyConsumer {
  // Called after the constructor has been called.
  #[activator]
  pub fn activate(&self, ts: &TidalService) {
    println!("MyConsumer Activated... {} - {:?}", ts.next_event(), self.tidal);
  }

  #[deactivator]
  pub fn deactivate(&self) {
    println!("MyConsumer Deactivated...");
  }

  #[update]
  pub fn update(&self) {
    println!("MyConsumer Updated.");
  }
}
```

The `#[activator]`, `#[deactivator]` and `#[update]` mark callback method
which are called when the service registration of the dependent service(s)
change. These are all optional.

* The `#[activator]` method is called when all injected services are present
and injected. It also provides a reference to the injected service object(s).
* The `#[deactivator]` method is called when not all required injected services
are available any more.
* The `#[update]` callback is made when the service registration properties
have changed.

## Example use from main method
Annotate the main method with `#[dynamic_services_main]`

```
#[dynamic_services_main]
fn main() {
  ...
```

Then register services like this:
```
  // Create the service
  let ts = TidalService {};

  // Set some service properties in a map
  let mut props = BTreeMap::new();
  props.insert("foo".to_owned(), "bar".to_owned());

  // Register the service and obtain the service registry.
  let sreg = register_service(Box::new(ts), props.clone());
```

You can update the service properties:
```
  props.insert("hi".to_string(), "ha".to_string());
  update_service(&sreg, props);
```

Finally you can un-register the service:
```
  unregister_service(sreg);
```
