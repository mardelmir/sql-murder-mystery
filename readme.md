# SQL Murder Mystery Walkthrough

## Solución con explicación
### 1. Petición para seleccionar el informe con las pistas iniciales
Originalmente, solo sabemos que existe un informe sobre el suceso con las siguientes características:
  - Se produjo en SQL City
  - La fecha del informe es 15 de enero de 2018
  - El tipo de crimen es asesinato

```SQL
SELECT * FROM crime_scene_report
WHERE date = 20180115 
AND type = 'murder'
AND city = 'SQL City'
```
La coincidencia devuelta indica que las cámaras de seguridad muestra que hubo dos testigos:
- El primero vive en la última casa de "Northwestern Dr"
- El segundo se llama Annabel y vive en una de las casas de "Franklin Ave"

### 2. Petición de búsqueda de los testigos y acceso a sus interrogatorios
JOIN de las tablas person (identificación de los testigos) e interview (transcrito de sus interrogatorios) para acceder a toda la información
```SQL
SELECT 
	person.id, 
	person.name,
	person.address_street_name,
	person.address_number,
	interview.transcript
FROM person
JOIN interview ON person.id = interview.person_id
WHERE address_street_name ='Northwestern Dr' -- Testigo 1: vive en la última casa (MAX()) de la calle Northwestern Dr
AND address_number = (SELECT MAX(address_number) FROM person WHERE address_street_name ='Northwestern Dr')
OR address_street_name = 'Franklin Ave' -- Testigo 2: viven en la calle Franklin Ave y su nombre es 'Annabel', no sabemos su apellido
AND name LIKE 'Annabel%'
```

### 3. Petición para encontrar al asesino
Tras leer los transcritos de los interrogatorios, la información aportada por los testigos es la siguiente:

Testigo 1 (Morty Schapiro):
  - El número de suscripción al gimnasio del asesino empieza por '48Z'
  - Este número es del grupo de los miembros 'gold'
  - La matrícula de su coche incluye 'H42W'

Testigo 2 (Annabel Miller):
  - Ha reconocido al asesino porque coincidió con él en el gimnasio el 9 de enero

Hacemos un JOIN de las tablas person, drivers_license, get_fit_now_member y get_fit_now_check_in para filtrar toda la información en una única consulta
```SQL
SELECT 
	person.id, person.name, person.license_id, 
	drivers_license.plate_number,
	get_fit_now_member.id,
	get_fit_now_check_in.check_in_date,
	get_fit_now_check_in.check_in_time,
	get_fit_now_check_in.check_out_time
FROM person
JOIN drivers_license ON person.license_id = drivers_license.id
JOIN get_fit_now_member ON person.id = get_fit_now_member.person_id
JOIN get_fit_now_check_in ON get_fit_now_member.id = get_fit_now_check_in.membership_id
WHERE plate_number LIKE '%H42W%' -- Tiene una matrícula como la que dice Morty
AND membership_id LIKE '48Z%' -- Tiene suscripción al gimnasio que empieza por 48Z
AND membership_status = 'gold' -- Miembro gold
OR person_id = 16371 -- Coincide con Annabel en el gimnasio
```

La única coincidencia devuelta es Jeremy Bowers, sin embargo en el mensaje de la solución nos indican que a Jeremy le han encargado la ejecución del asesinato y que no es el verdadero cerebro de la operación. Si queremos saber quién está detrás de todo necesitamos acceder al interrogatorio de Jeremy

### Petición para acceder al interrogatorio de Jeremy e información sobre el verdadero asesino
```SQL
SELECT * FROM interview
WHERE person_id = 67318
```
Del transcrito de este interrogatorio sabemos que:
  - La responsable es una mujer con mucho dinero, nombre desconocido
  - Mide entre 5'5" (65") y 5'7" (67")
  - Es pelirroja
  - Conduce un Tesla Model S
  - Ha asistido al SQL Symphony Concert 3 veces en diciembre de 2017



### Petición para identificar al verdadero asesino
```SQL
SELECT 
	person.id,
	person.name,
	drivers_license.height,
	drivers_license.hair_color,
	drivers_license.car_make,
	drivers_license.car_model,
	income.annual_income
FROM person
JOIN drivers_license ON person.license_id = drivers_license.id
JOIN income ON person.ssn = income.ssn
JOIN facebook_event_checkin ON person.id = facebook_event_checkin.person_id
WHERE height BETWEEN 65 AND 67 -- Altura entre 5'5" y 5'6"
AND hair_color = 'red' -- Pelirroja
AND car_make = 'Tesla' -- Marca del coche
AND car_model = 'Model S' -- Modelo del coche
AND date BETWEEN 20171201 AND 20171231 -- Asiste 3 veces al SQL Symphony Concert en diciembre 2017
AND event_name = 'SQL Symphony Concert'
GROUP BY person_id
HAVING COUNT(person_id) = 3
```

Esta búsqueda devuelve una única coincidencia: Miranda Priestly
