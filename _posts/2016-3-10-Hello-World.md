---
layout: post
title: Solución al problema de consultas por ubicación con Firebase y Geofire
---
El principal problema o inconveniente que presenta Firebase dentro de todas las facilidades que nos proporciona, está en las limitaciones al momento de realizar consultas con su base de datos. La falta de consultas por más de un atributo es todo un cambio de paradigma para aquellos no conocedores de las bases de datos NOSQL. 

De cualquier forma, dentro del proyecto que estoy desarrollando [(Tobara)](https://tobarapp.github.io), me encontré con la necesidad de obtener información en base a su ubicación. Lamentablemente, al utilizar un *FirebaseRecyclerAdapter* y no tener tiempo ni ganas de construir mi propio adaptador, me enconté con que no podía realizar filtrado de la query antes de poblarlo (El adaptador se llena a través de una *Query* por lo que no se asignan elementos mediante *Arrays* o *Listas*). 

Estuve un tiempo tratando de pensar como solucionarlo hasta el punto de asumir que había perdido y que tendría que implementar mi adaptador desde cero. Afortunadamente, revisando información por internet llegué al blog de *Google* donde encontré este [link](https://firebase.googleblog.com/2013/04/denormalizing-your-data-is-normal.html) que me hizo entender mejor como funciona Firebase y su Base de datos. La clave es entender que a diferencia de las bases de datos relacionales, es necesario denormalizar los datos para poder realizar consultas de manera eficiente. 

Finalmente, gracias a esta información entendí que es necesario en ciertos casos, duplicar o replicar información relevante de algunos nodos en pos de eficiencia operativa y manejo de datos. Conociendo esta información, me decidí por replicar la información de cada usuario en base a su ubicación y solo modificarla cuando el usuario abandona la ubicación o radio preestablecido por *Geofire*. 

Un extracto de la implementación la pueden ver a continuación: 
```java 
public void getPostbyLocation(){
        GeoFire geoFire = new GeoFire(mDatabase.child("post_locations"));
        GPSTracker gps = new GPSTracker(getContext());
        if(gps.canGetLocation()){
            double latitude = gps.getLatitude();
            double longitude = gps.getLongitude();
            GeoQuery geoQuery = geoFire.queryAtLocation(new GeoLocation(latitude,longitude), 20);
            geoQuery.addGeoQueryEventListener(new GeoQueryEventListener() {
                @Override
                public void onKeyEntered(final String key, GeoLocation location) {
                    mDatabase.child("posts").child(key).addListenerForSingleValueEvent(new ValueEventListener() {
                        @Override
                        public void onDataChange(DataSnapshot dataSnapshot) {
                            Post p = dataSnapshot.getValue(Post.class);
                            Map<String, Object> postValues = p.toMap();
                            Map<String, Object> childUpdates = new HashMap<>();
                            childUpdates.put("/posts_locations_by_user/"+ getUid() +"/"+ key, postValues);
                            mDatabase.updateChildren(childUpdates);
                        }
                        @Override
                        public void onCancelled(DatabaseError databaseError) {}
                    });
                    System.out.println(String.format("Key %s entered the search area at [%f,%f]", key, location.latitude, location.longitude));
                }
```

```java
                @Override
                public void onKeyExited(String key) {
                    mDatabase.child("posts_locations_by_user").removeValue();
                    System.out.println(String.format("Key %s is no longer in the search area", key));
                }

                @Override
                public void onKeyMoved(String key, GeoLocation location) {
                    System.out.println(String.format("Key %s moved within the search area to [%f,%f]", key, location.latitude, location.longitude));
                }

                @Override
                public void onGeoQueryReady() {
                    System.out.println("All initial data has been loaded and events have been fired!");
                }

                @Override
                public void onGeoQueryError(DatabaseError error) {
                    System.err.println("There was an error with this query: " + error);
                }
            });
        }else{
            // can't get location
            // GPS or Network is not enabled
            // Ask user to enable GPS/network in settings
            gps.showSettingsAlert();
        }
    }
```
Podemos ver que defino una *Geoquery (Geofire Query)* en base a mi ubicación, para que encuentre los *posts* más cercanos en base a mi ubicación en un radio de 20 *km*. De esta forma cuando encuentra los *posts* en *OnKeyEntered()*, lo que hago es crear un nuevo nodo, que se identifique por el *uid* del usuario para almacenar los posts cercanos. De esta forma, los posts cercanos se mantienen almacenados solo mientras el usuario se mantenga en ese radio; al salir de éste, en el método *OnKeyExited*, se eliminan los nodos anteriores por el resultado de la nueva *Geo Query*. 

De esta forma, finalmente puedo obtener la información que necesitaba sin necesitar crear un adaptador ni limitarme por las convenciones que presenta el adaptador de Firebase. 
 
