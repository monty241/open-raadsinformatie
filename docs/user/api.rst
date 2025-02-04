.. _restapi:

RESTful API
===========

.. warning::

   This page currently shows a draft of the API specification. **The format of some of the request/response pairs is still subject to change!**

General notes
-------------

The API accepts requests with JSON content and returns JSON data in all of its responses (unless stated otherwise). Standard HTTP response codes are used to indicate errors. In case of an error, a more detailed description can be found in the JSON response body. UTF-8 character encoding is used in both requests and responses.

All API URLs referenced in this documentation start with the following base part:

    :rest_api_v0:`v0`

All API endpoints are designed according to the idea that there is an operation within a *context*: methods on the "root" context are executed across all datasets; :ref:`/search <rest_search>` executes a search across all data sources, whereas :ref:`/amstelveen/search <rest_source_search>` executes a search on the Amstelveen data source.

Arguments to an endpoint are placed behind the method definition, or supplied as JSON in a POST request. For instance, the :ref:`similar objects endpoint <rest_similar>` can be executed within the context of a collection, and needs an ``object_id`` to execute on.

Collection overview and statistics
----------------------------------

.. http:get:: /sources

   Get a list of all available sources (collections) with item counts

   :statuscode 200: OK, no errors.

   **Example request**

   .. sourcecode:: http

      $ curl -i -XGET 'http://api.openraadsinformatie.nl/v0/sources'

   **Example response**

    .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-type: application/json
      Content-length: 369
      Date: Wed, 13 Oct 2015 12:36:15 GMT

      {
        "sources": [
          {
            "events": 1026,
            "id": "combined_index",
            "organizations": 66,
            "persons": 233
          },
          {
            "events": 607,
            "id": "amstelveen",
            "organizations": 15,
            "persons": 54
          },
          {
            "events": 265,
            "id": "den_helder",
            "organizations": 27,
            "persons": 50
          },
          {
            "events": 132,
            "id": "heerde",
            "organizations": 11,
            "persons": 30
          },
          {
            "id": "utrecht",
            "organizations": 16,
            "persons": 64
          },
          {
            "events": 22,
            "id": "oude_ijsselstreek",
            "organizations": 18,
            "persons": 35
          }
        ]
      }

.. _rest_queryparams:

Query parameters
----------------

The POST requests listed below accept query parameters:

**Query**

Besides standard keyword searches, a basic query syntax is supported. This syntax supports the following special characters:

- ``+`` signifies an AND operation

- ``|`` signifies an OR operation
- ``-`` negates a single token
- ``"`` wraps a number of tokens to signify a phrase for searching
- ``*`` at the end of a term signifies a prefix query
- ``(`` and ``)`` signify precedence

The default strategy is to perform an AND query.

**Facets**

.. warning:: In Elasticsearch 5.x the API changed, replacing ``facets`` by ``aggregations``. For compatibility reasons facets will still be used in this project.

The ``facets`` object determines which facets should be returned. The keys of this object should contain the names of a the requested facets, the values should be objects. These objects are used to set per facet options. Facet defaults will be used when the options dictionary is empty.

To specify the number of facet values that should be returned (for term based facets):

.. sourcecode:: javascript

  "facets": {
     "classification": {"size": 3}
  }

For a date based facet the 'bucket size' of the histogram can be specified:

.. sourcecode:: javascript

  "facets": {
     "start_date": {"interval": "year"}
  }

Allowed sizes are ``year``, ``quarter``, ``month``, ``week`` and ``day`` (the default size is ``month``).

**Filters**

Results can be filtered on one or more properties. Each key of the ``filters`` object represents a filter, the values should be objects. When filtering on multiple fields only documents that match all filters are included in the result set. The names of the filters match those of the facets:

.. sourcecode:: javascript

  "filters": {
     "classification": {
        "terms": ["Agenda", "Agendapunt"]
     }
  }

Use the following format to filter on a date range:

.. sourcecode:: javascript

  "filters": {
     "date": {
        "from": "2011-12-24",
        "to": "2011-12-28"
     }
  }


**Available filters and facets**

Each doc_type can have different fields to ``filters`` or make ``facets`` on:

+-------------------+--------------------------------+------------------------+
| Doc_type          | Field                          | Type                   |
+===================+================================+========================+
| <all>             | processing_started             | date_histogram         |
|                   +--------------------------------+------------------------+
|                   | processing_finished            | date_histogram         |
|                   +--------------------------------+------------------------+
|                   | source                         | terms                  |
|                   +--------------------------------+------------------------+
|                   | collection                     | terms                  |
|                   +--------------------------------+------------------------+
|                   | rights                         | terms                  |
|                   +--------------------------------+------------------------+
|                   | index                          | terms                  |
|                   +--------------------------------+------------------------+
|                   | types                          | terms                  |
|                   +--------------------------------+------------------------+
|                   | start_date                     | date_histogram         |
+-------------------+--------------------------------+------------------------+
| <empty>           | classification                 | terms                  |
+-------------------+--------------------------------+------------------------+
| events            | classification                 | terms                  |
|                   +--------------------------------+------------------------+
|                   | organization_id                | terms                  |
|                   +--------------------------------+------------------------+
|                   | location                       | terms                  |
|                   +--------------------------------+------------------------+
|                   | status                         | terms                  |
|                   +--------------------------------+------------------------+
|                   | start_date                     | date_histogram         |
|                   +--------------------------------+------------------------+
|                   | end_date                       | date_histogram         |
+-------------------+--------------------------------+------------------------+
| vote_events       | classification                 | terms                  |
|                   +--------------------------------+------------------------+
|                   | organization_id                | terms                  |
|                   +--------------------------------+------------------------+
|                   | start_date                     | date_histogram         |
|                   +--------------------------------+------------------------+
|                   | end_date                       | date_histogram         |
|                   +--------------------------------+------------------------+
|                   | legislative_session_id         | terms                  |
+-------------------+--------------------------------+------------------------+
| motions           | classification                 | terms                  |
|                   +--------------------------------+------------------------+
|                   | organization_id                | terms                  |
|                   +--------------------------------+------------------------+
|                   | legislative_session_id         | terms                  |
|                   +--------------------------------+------------------------+
|                   | creator_id                     | terms                  |
|                   +--------------------------------+------------------------+
|                   | date                           | date_histogram         |
|                   +--------------------------------+------------------------+
|                   | requirement                    | terms                  |
|                   +--------------------------------+------------------------+
|                   | result                         | terms                  |
+-------------------+--------------------------------+------------------------+
| persons           | gender                         | terms*                 |
|                   +--------------------------------+------------------------+
|                   | organization                   | terms                  |
+-------------------+--------------------------------+------------------------+
| organizations     | classification                 | terms                  |
+-------------------+--------------------------------+------------------------+

Fields with class ``<all>`` can be used with all doc_types.
Fields with class ``<empty>`` can only be used when no doc_type is specified, although ``<all>`` fields can be used as well.

Types that are ``terms`` have a default ``size`` of 10, except for ``gender`` with a default size of 2.
The ``date_histogram`` type has a default ``interval`` of one month.

**A combination of filters and facets might look like this:**

.. sourcecode:: javascript

  "filters": {
     "classification": {
        "terms": ["Agenda", "Agendapunt"]
     },
     "processing_started": {
        "from": "2016-12-24",
        "to": "2017-12-28"
     }
  },
  "facets": {
     "collection": {},
     "classification": {
         "size": 2
     },
     "start_date": {
         "interval": "month"
     }
  }

.. _rest_search:

Searching within multiple collections
-------------------------------------

.. http:post:: /search
.. http:post:: /search/(doc_type)

   Search for items through all indexed datasets. The search can be limited on a certain document type. `doc_type` can assume the following values:

   * persons
   * organizations
   * events
   * motions
   * vote_events

   :jsonparameter query: one or more keywords.
   :jsonparameter filters: an object with field and values to filter on (optional).
   :jsonparameter facets: an object with fields for which to return facets (optional).
   :jsonparameter sort: the field the search results are sorted on. By default, results are sorted by relevancy to the query.
   :jsonparameter size: the maximum number of documents to return (optional, defaults to 10).
   :jsonparameter from: the offset from the first result (optional, defaults to 0).
   :statuscode 200: OK, no errors.
   :statuscode 400: Bad Request. An accompanying error message will explain why the request was invalid.

   **Example request**

   .. sourcecode:: http

      $ curl -i -XPOST 'http://api.openraadsinformatie.nl/v0/search' -d '{
         "query": "vergadering",
         "facets": {
            "collection": {},
            "start_date": {"interval": "year"}
         },
         "size": 1
      }'

   **Example response**

    .. sourcecode:: http

        HTTP/1.1 200 OK
        Server: nginx/1.4.6 (Ubuntu)
        Date: Mon, 29 Jan 2018 17:54:39 GMT
        Content-Type: application/json
        Content-Length: 17576
        Connection: keep-alive
        Access-Control-Allow-Origin: *
        Access-Control-Allow-Headers: origin, content-type, accept

        {
          "events": [
            {
              "children": [
                "068b3868ffac5380cf72c5e1aff88560144c26d8"
              ],
              "classification": "Agenda",
              "id": "5f98609d308559ebea67f4c67fff098d8381d30f",
              "identifiers": [
                {
                  "identifier": "873",
                  "scheme": "GemeenteOplossingen"
                },
                {
                  "identifier": "5f98609d308559ebea67f4c67fff098d8381d30f",
                  "scheme": "ORI"
                }
              ],
              "location": "Raadzaal",
              "meta": {
                "_index": "ori_combined_index_new_migrated",
                "_score": 43.942627,
                "_type": "events",
                "collection": "steenbergen",
                "highlight": {
                  "name": [
                    "Beeldvormende <em>vergadering</em>"
                  ]
                },
                "ocd_url": "http://api.openraadsinformatie.nl/v0/steenbergen_meetings/5f98609d308559ebea67f4c67fff098d8381d30f",
                "original_object_id": "873",
                "original_object_urls": {
                  "html": "https://raad.gemeente-steenbergen.nl/api/v1/meetings/873"
                },
                "processing_finished": "2018-01-28T10:44:36.062902",
                "processing_started": "2018-01-28T10:44:32.919763",
                "rights": "undefined",
                "source_id": "steenbergen_meetings"
              },
              "name": "Beeldvormende vergadering",
              "organization": {
                "classification": "committee",
                "description": "Beeldvormende vergadering",
                "id": "1ef91eeb4ef535dc1ba55ebcd9543f90533d3ce3",
                "identifiers": [
                  {
                    "identifier": "1ef91eeb4ef535dc1ba55ebcd9543f90533d3ce3",
                    "scheme": "ORI"
                  },
                  {
                    "identifier": 31,
                    "scheme": "GemeenteOplossingen"
                  }
                ],
                "meta": {
                  "_index": "ori_steenbergen_20170207165350_migrated",
                  "_score": 1.0,
                  "_type": "organizations",
                  "collection": "steenbergen",
                  "ocd_url": "http://frontend:5000/v0/steenbergen_committees/1ef91eeb4ef535dc1ba55ebcd9543f90533d3ce3",
                  "original_object_id": "31",
                  "original_object_urls": {
                    "html": "https://raad.gemeente-steenbergen.nl/api/v1/dmus"
                  },
                  "processing_finished": "2018-01-28T10:44:27.743148",
                  "processing_started": "2018-01-28T10:44:27.119217",
                  "rights": "undefined",
                  "source_id": "steenbergen_committees"
                },
                "name": "Beeldvormende vergadering",
                "sources": [
                  {
                    "note": "",
                    "url": "https://raad.gemeente-steenbergen.nl/api/v1/dmus"
                  }
                ]
              },
              "organization_id": "31",
              "sources": [
                {
                  "description": "*BM1501087*\nBM1501087\nVerslag\nDatum\nTijd\nLocatie\nOnderwerp\n:\n:\n:\n:\n:\nBeeldvormende vergadering\n9 april 2015\n19:30 uur- 22:00 uur\nDorpshuis De Vaert\nCentrumplan Welberg\nAanwezig\nAfwezig\n:\n:\nzie lijst\nVoor de pauze wordt er in het algemeen gevraagd wat voor gedachten er leven over het nieuwe\ndorpshart. De kinderen van de Puis X vragen in een presentatie aandacht voor de verkeersveiligheid en\ndragen al een concrete oplossing voor. Vanuit de zaal wordt gevraagd om de verkeerssituatie rondom de\nschool al sneller op te pakken niet te wachten tot 2018.\nDe volgende hoofdlijnen komen uit de discussie naar voren:\nVerkeer:\nBijna iedereen is het erover eens dat de verkeersituatie dient te verbeteren door het aanleggen van fiets-\nen voetpaden het autovrij maken van een groot gedeelte van het gebied en een nieuwe ontsluiting te\ncre\u00ebren voor de auto\u2019s.\nDe Vaert en andere clubgebouwen\nDe gebruikers van de Vaert zijn over het algemeen tevreden met de huidige accommodatie. Hetzelfde\ngeld voor de clubgebouwen van de JFK en MJW de Klup. Wel zou het pand, de panden gemoderniseerd\nkunnen worden. Ondanks de tevredenheid van de gebruikers wordt er door een gedeelte van de\naanwezigen geopperd om de kerk te gebruiken als nieuw dorpshuis en het huidige dorpshuis dan af te\nbreken. Duidelijk blijkt dat er verdeeldheid is over dit voorstel, waarbij vooral het kostenaspect centraal\nkomt te staan. Om hier duidelijkheid over te krijgen wordt het voorstel gedaan om de kosten door te\nberekenen van beide opties om zo een goed oordeel te kunnen vormen over de haalbaarheid en de\ngevolgen voor de gebruikers.\nKerk:\nDe aanwezigen willen de kerk in het liefst in zijn geheel behouden, mocht dit niet mogelijk zijn dan moet in\nieder geval de kerktoren behouden blijven. Het kerkbestuur staat open voor een\nculturele/maatschappelijke invulling van de kerk.\nCentrumplan te klein\nEen aantal aanwezigen is van mening dat het centrumgebied te klein is en er ruimer dient te worden\ngekeken. Het staat de aanwezigen vanzelfsprekend vrij om ook andere zaken in te brengen die aandacht\nbehoeven en niet in het directe aangewezen centrumgebied zijn gelegen.\nBoomvaart\nDe Boomvaart dient meer betrokken te worden bij het centrum door de Boomvaart en dient een meer\nopen karakter te krijgen ( Nota bene: Een voorstel tot overkluizing van de Boomvaart wordt door een grote\nmeerderheid afgewezen.).\nOverige openbare ruimte\nIn het centrumgebied dient ruimte gecre\u00eberd te worden voor een trap/speelveldje.\nWoningbouw\nOp een gedeelte van terrein kunnen woningen gebouwd worden die kunnen dienen als kostendekker voor\nde investeringen.\nSchool:\nDe school dient gehandhaafd te blijven op de huidige locatie.\nWinkels:\nEen meerderheid acht het niet realistisch en noodzakelijk dat er winkels worden opgericht in de Welberg.\nNa de pauze worden er vier groepen ingedeeld voor verdere uitwerking. 2 groepen waarbij de\nmaatschappelijke voorzieningen gebruik blijven maken van het huidige dorpshuis en 2 waarbij het huidige\ndorpshuis niet gehandhaafd blijft.\nGroep 1\nVerkeer\nIn de Kapelaan Kockstraat dienen snelheidsremmers te worden aangebracht.\nNieuwe ontsluiting naar school achter de kerk langs.\nGemeenschapshuis De Vaert\nGemeenschapshuis De Vaert verdwijnt. Een nieuw gemeenschapshuis wordt gebouwd op het terrein.\nWaardoor er een grotere aaneengesloten openbare ruimte overblijft.\nKerk\nDe Kerk wordt gesloopt met uitzondering van de kerktoren. De kerktoren blijft behouden.\nBoomvaart\n2\nTer hoogte van de kerk wordt de Boomvaart gedeeltelijk overkluisd om verkeerssituatie te verbeteren. Ter\nhoogte van het parkeerterrein en het huidige dorpshuis wordt de Vaart juist groter gemaakt (Welberg aan\nZee).\nOverige openbare ruimte\nOp de huidige locatie van de kerk worden speeltoestellen geplaatst.\nWoningbouw/openbare ruimte\nEr dienen een aantal seniorenwoningen en starterswoningen gebouwd te worden in een parkachtige\nomgeving op het resterende terrein.\nSchool\nDe school blijft gehandhaafd.\nOverige\nHet terrein naast de begraafplaats gebruiken als veld om as te strooien en om een gedenkbos te strooien.\nGroep 2\nVerkeer:\nAchter en gedeeltelijk op de huidige locatie van de Vaert parkeerterrein inrichten. Gedeeltelijk ook\nparkeren onderbrengen op terrein naast de begraafplaats.\nFietsers kunnen achterlangs langs de kerk naar school, via de Corneliusstraat.\n(plan A) Brede ontsluiting realiseren voor auto\u2019s door \u00e9\u00e9n grotere ontsluiting te cre\u00ebren via de Pastoor\nKerckestraat. Bestaand huis dient dan te worden afgebroken.\n(plan B) Rotonde realiseren bij Kapelaan Kockstraat, stukje van de Vaert overkluizen om mogelijk te\nmaken en dan ontsluiting langs de kerk realiseren.\nGemeenschapshuis De Vaert\nDe Vaert wordt gesloopt. De functies die nu in De Vaert zijn, worden ondergebracht in de kerk.\nKerk:\nDe kerk in zijn geheel handhaven en gebruiken als multifunctioneel centrum, evt. ook met appartementen.\nBoomvaart\nDe Boomvaart breder en ruimer maken.\nOverige openbare ruimte:\n3\nSpeelplaats maken achter de school en Kerk.\nGedeelte gelegen, naast de Boomvaart, op het huidige parkeerterrein en voor een groot gedeelte op de\nhuidige locatie van De Vaert park aanleggen.\nWoningbouw\nOp het voormalige voetbalveld energie neutrale woningen bouwen.\nKosten:\nKosten gebruik en onderhoud huidige dorpshuis vergelijken met nieuwe situatie en ook kijken naar evt.\nsubsidiemogelijkheden.\nGroep 3:\nVerkeer:\nVoorstel kinderen Pius X uitwerken met twee aanvullingen: 1: Een bredere weg met fietspad maken langs\nde Boomvaart en 2 Fiets/voetpad doortrekken achter de kerk naar de Corneliusstraat\nVrachtwagens buiten het centrum houden.\nGemeenschapshuis De Vaert:\nGemeenschapshuis de Vaert blijft op zijn huidige locatie. Het is wenselijk om de Vaert beperkt uit te\nbreiden.\nKerk:\nIn de kerk kunnen appartementen worden gerealiseerd.\nOverige openbare ruimte:\nDe speeltuin wordt verplaatst naar het oude sportveld. Het speelterrein/trapveldje komt definitief op het\noude sportveld.\nDe containers die voor de Vaert staan zouden verplaatst moeten worden naar de Laurentiusdijk\nBoomvaart:\nHandhaven zoals nu het geval.\nWoningbouw:\nAlleen in de kerk appartementen, verder niet.\nGroep 4:\nVerkeer\nRechtstreekse aansluiting van Corneliusstraat achter de kerk naar de Pastoor Kerckestraat.\nFietspaden realiseren naar de school.\nVoetpaden aanleggen in centrumgebied.\nParkeerplaatsen voor de kerk weghalen. Hier kan een dorpsplein worden gerealiseerd.\nVoetgangersoversteekplaats realiseren bij de Corneliusstraat.\nGemeenschapshuis De Vaert:\nGemeenschapshuis De Vaert openhouden, dus handhaven.\nKerk\nDe kerk in zijn geheel handhaven en hier betaalbare appartementen in maken.\nBoomvaart\nDe Boomvaart moet open blijven.\nOverige openbare ruimte\n4\nPlein cre\u00ebren bij kern door parkeren hier weg te halen en bomen en bankjes te plaatsen.\nBomen planten langs de gehele Boomvaart binnen de bebouwde kom van Welberg\nHet hele dorp gezelliger aankleden.\nVoormalige sportveld gebruiken als speelveld zoals nu het geval. Ook MJW de Klup en de JFK kunnen\nhier gebruik van maken.\nEr dient meer toezicht beplanting en verlichting te worden gerealiseerd op het terrein van het voormalige\nsportveld.\nHet terrein bij de school dient ook door de clubs te kunnen worden gebruikt\nWoningbouw:\nMogelijk naast woningbouw in kerk in beperkte mate bebouwing toelaten op voormalig sportveld.\nSchool\nDe school blijft gehandhaafd.\n5",
                  "note": "Verslag beeldv vergadering 090415.pdf",
                  "url": "https://raad.gemeente-steenbergen.nl/api/v1/meetings/873/documents/5184"
                }
              ],
              "start_date": "2015-04-09T19:30:00+00:00",
              "status": "confirmed"
            }
          ],
          "facets": {
            "collection": {
              "buckets": [
                {
                  "doc_count": 7749,
                  "key": "amsterdam"
                },
                {
                  "doc_count": 6590,
                  "key": "utrecht"
                },
                {
                  "doc_count": 4435,
                  "key": "katwijk"
                },
                {
                  "doc_count": 4082,
                  "key": "medemblik"
                },
                {
                  "doc_count": 2714,
                  "key": "zoetermeer"
                },
                {
                  "doc_count": 2620,
                  "key": "zwolle"
                },
                {
                  "doc_count": 2561,
                  "key": "den_helder"
                },
                {
                  "doc_count": 2543,
                  "key": "hilversum"
                },
                {
                  "doc_count": 2178,
                  "key": "maassluis"
                },
                {
                  "doc_count": 2012,
                  "key": "amstelveen"
                }
              ],
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 77977,
              "terms": [
                {
                  "count": 7749,
                  "term": "amsterdam"
                },
                {
                  "count": 6590,
                  "term": "utrecht"
                },
                {
                  "count": 4435,
                  "term": "katwijk"
                },
                {
                  "count": 4082,
                  "term": "medemblik"
                },
                {
                  "count": 2714,
                  "term": "zoetermeer"
                },
                {
                  "count": 2620,
                  "term": "zwolle"
                },
                {
                  "count": 2561,
                  "term": "den_helder"
                },
                {
                  "count": 2543,
                  "term": "hilversum"
                },
                {
                  "count": 2178,
                  "term": "maassluis"
                },
                {
                  "count": 2012,
                  "term": "amstelveen"
                }
              ]
            },
            "start_date": {
              "buckets": [
                {
                  "doc_count": 1,
                  "key": 788918400000,
                  "key_as_string": "1995-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 1,
                  "key": 820454400000,
                  "key_as_string": "1996-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 24,
                  "key": 852076800000,
                  "key_as_string": "1997-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 40,
                  "key": 883612800000,
                  "key_as_string": "1998-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 17,
                  "key": 915148800000,
                  "key_as_string": "1999-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 2,
                  "key": 946684800000,
                  "key_as_string": "2000-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 21,
                  "key": 978307200000,
                  "key_as_string": "2001-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 38,
                  "key": 1009843200000,
                  "key_as_string": "2002-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 103,
                  "key": 1041379200000,
                  "key_as_string": "2003-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 141,
                  "key": 1072915200000,
                  "key_as_string": "2004-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 168,
                  "key": 1104537600000,
                  "key_as_string": "2005-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 182,
                  "key": 1136073600000,
                  "key_as_string": "2006-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 170,
                  "key": 1167609600000,
                  "key_as_string": "2007-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 212,
                  "key": 1199145600000,
                  "key_as_string": "2008-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 302,
                  "key": 1230768000000,
                  "key_as_string": "2009-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 302,
                  "key": 1262304000000,
                  "key_as_string": "2010-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 606,
                  "key": 1293840000000,
                  "key_as_string": "2011-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 1377,
                  "key": 1325376000000,
                  "key_as_string": "2012-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 2145,
                  "key": 1356998400000,
                  "key_as_string": "2013-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 11789,
                  "key": 1388534400000,
                  "key_as_string": "2014-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 13571,
                  "key": 1420070400000,
                  "key_as_string": "2015-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 15857,
                  "key": 1451606400000,
                  "key_as_string": "2016-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 19708,
                  "key": 1483228800000,
                  "key_as_string": "2017-01-01T00:00:00.000Z"
                },
                {
                  "doc_count": 2492,
                  "key": 1514764800000,
                  "key_as_string": "2018-01-01T00:00:00.000Z"
                }
              ],
              "terms": [
                {
                  "count": 1,
                  "term": 788918400000
                },
                {
                  "count": 1,
                  "term": 820454400000
                },
                {
                  "count": 24,
                  "term": 852076800000
                },
                {
                  "count": 40,
                  "term": 883612800000
                },
                {
                  "count": 17,
                  "term": 915148800000
                },
                {
                  "count": 2,
                  "term": 946684800000
                },
                {
                  "count": 21,
                  "term": 978307200000
                },
                {
                  "count": 38,
                  "term": 1009843200000
                },
                {
                  "count": 103,
                  "term": 1041379200000
                },
                {
                  "count": 141,
                  "term": 1072915200000
                },
                {
                  "count": 168,
                  "term": 1104537600000
                },
                {
                  "count": 182,
                  "term": 1136073600000
                },
                {
                  "count": 170,
                  "term": 1167609600000
                },
                {
                  "count": 212,
                  "term": 1199145600000
                },
                {
                  "count": 302,
                  "term": 1230768000000
                },
                {
                  "count": 302,
                  "term": 1262304000000
                },
                {
                  "count": 606,
                  "term": 1293840000000
                },
                {
                  "count": 1377,
                  "term": 1325376000000
                },
                {
                  "count": 2145,
                  "term": 1356998400000
                },
                {
                  "count": 11789,
                  "term": 1388534400000
                },
                {
                  "count": 13571,
                  "term": 1420070400000
                },
                {
                  "count": 15857,
                  "term": 1451606400000
                },
                {
                  "count": 19708,
                  "term": 1483228800000
                },
                {
                  "count": 2492,
                  "term": 1514764800000
                }
              ]
            }
          },
          "meta": {
            "took": 39,
            "total": 115461
          }
        }

.. _rest_source_search:

Searching within a single collection
------------------------------------


.. http:post:: /(source_id)/search
.. http:post:: /(source_id)/(doc_type)/search

   Search for objects within a specific dataset. The objects returned by this method may also include fields that are specific to the queried dataset, rather than only those fields that all indexed datasets have in common. The search can be restricted to a certain `doc_type`, in the same way as the previous API call does.

   See specifications of the :ref:`search method <rest_search>` for the request and response format.

   :jsonparameter query: one or more keywords.
   :jsonparameter filters: an object with field and values to filter on (optional).
   :jsonparameter facets: an object with fields for which to return facets (optional).
   :jsonparameter sort: the field the search results are sorted on. By default, results are sorted by relevancy to the query.
   :jsonparameter size: the maximum number of documents to return (optional, defaults to 10).
   :jsonparameter from: the offset from the first result (optional, defaults to 0).
   :statuscode 200: OK, no errors.
   :statuscode 400: Bad Request. An accompanying error message will explain why the request was invalid.
   :statuscode 404: The requested source does not exist.

.. _rest_scroll:

Searching with scroll
---------------------


.. http:post:: /(source_id)/search
.. http:post:: /(source_id)/(doc_type)/search

   Search for objects within a specific dataset. The objects returned by this method may also include fields that are specific to the queried dataset, rather than only those fields that all indexed datasets have in common. The search can be restricted to a certain `doc_type`, in the same way as the previous API call does.
   A scroll cursor is returned so you can get all results (Due to handling in Elasticsearch general paging is limited to 10.000 items).
   Scrolling works akin to Elasticsearch scrolling, except that the first call which a scroll parameter also returns the first page of results. Subsequent calls need only have the scroll and scroll_id parameters and will return subsequent pages.
   The scroll id is returned in the meta dictionary returned as part of the results.

   See specifications of the :ref:`search method <rest_search>` for the request and response format.

   :jsonparameter query: one or more keywords.
   :jsonparameter filters: an object with field and values to filter on (optional).
   :jsonparameter facets: an object with fields for which to return facets (optional).
   :jsonparameter sort: the field the search results are sorted on. By default, results are sorted by relevancy to the query.
   :jsonparameter size: the maximum number of documents to return (optional, defaults to 10).
   :jsonparameter scroll: the scroll window (eg. 1m).
   :jsonparameter scroll_id: the scroll cursor which was returned in previous calls.
   :statuscode 200: OK, no errors.
   :statuscode 400: Bad Request. An accompanying error message will explain why the request was invalid.
   :statuscode 404: The requested source does not exist.


.. _rest_get:

Retrieving a single object
--------------------------

.. http:get:: /(source_id)/(doc_type)/(object_id)

   Retrieve the contents of a single object. The `doc_type` needs to be specified.

   :statuscode 200: OK, no errors.
   :statuscode 404: The source and/or object does not exist.

   **Example request**

   .. sourcecode:: http

      $ curl -i 'http://api.openraadsinformatie.nl/v0/den_helder/events/4a39c497c7818af9ad6d7d8335a6e951d6a83bd6'

   **Example response**

   .. sourcecode:: http

      HTTP/1.1 200 OK
      content-type: application/json
      content-length: 3167
      date: Wed, 27 May 2015 12:55:04 GMT

      {
        "children": [
          "5995a9ba50baac60f6a7e3ac0c14c1a721f8103d",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "8f48d0c951293094c1168bde1e9e563f05fd419f",
          "b0a92dd85444b66a9881187c0d48c557d139817b",
          "fd789cf90f664a69f6ce6a94f9d536dff84e405c",
          "6e91e7eda058df0d15de6a92ba930947e41c0d4f",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "5f34b92921d733498e31f430d61e6683dfea23d0",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "89f2d80e2da9206ee6b0239ed5255e63be053f32",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "b6bf2d51f5c91a7709573a894ab65845ac1110e4",
          "6f15e189b3ad8165f3b23520dc61287ae16ac3df"
        ],
        "classification": "Meeting",
        "end_date": "2015-10-12T19:30:00",
        "id": "4a39c497c7818af9ad6d7d8335a6e951d6a83bd6",
        "identifiers": [
          {
            "identifier": "https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/",
            "scheme": "GemeenteOplossingen"
          },
          {
            "identifier": "4a39c497c7818af9ad6d7d8335a6e951d6a83bd6",
            "scheme": "ORI"
          }
        ],
        "location": "Gemeentehuis",
        "meta": {
          "collection": "Gemeenteraad 12 oktober 2015 19:30:00",
          "original_object_id": "https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/00:00/",
          "original_object_urls": {
            "html": "https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/"
          },
          "processing_finished": "2015-10-12T15:11:06.400560",
          "processing_started": "2015-10-12T15:07:26.693176",
          "rights": "undefined",
          "source_id": "den_helder_meetings"
        },
        "name": "Gemeenteraad 12 oktober 2015 19:30:00",
        "organization": {
          "classification": "Council",
          "description": "Gemeente Den Helder (Den Helder) (NH)",
          "id": "gemeente-den-helder-den-helder-nh",
          "identifiers": [
            {
              "id": "56166f5b9e5d9edf0065a84d",
              "identifier": "gemeente-den-helder-den-helder-nh",
              "scheme": "ORI"
            }
          ],
          "memberships": [
            {
              "id": "e71d0389791f617a0318acd65e882e59",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "c540b876c70fcf3a8e90474842d8221eecf0c78a",
              "role": "Raadslid"
            },
            {
              "id": "87952487de4dc138cd09da0d04be73c2",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "1f02666f7d7a3ad6ea8cf63c429fdff1d6f39045",
              "role": "Raadslid"
            },
            {
              "id": "c9d325d8d548a3fb3203a851441447b0",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "067affd24368c5877cec0d0cab8dd8941647d9e3",
              "role": "Raadslid"
            },
            {
              "id": "e57e00b57cddee57558f535364e4860a",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "ab3b30fa8c94b4b616e038069c6a08bf0ba43a1f",
              "role": "Fractievoorzitter"
            },
            {
              "id": "80fd1c29a80954a0bca704f28452fc6f",
              "label": "Wethouder",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "9189b630d442ec7e48decc50492b8e732b3f4568",
              "role": "Wethouder"
            },
            {
              "id": "ecd404f4686a03f44949090a1941670d",
              "label": "Raadsgriffier",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "d3228dc2e838235efba39cefc1bfa92c12cd7f60",
              "role": "Raadsgriffier"
            },
            {
              "id": "b5f687d322be7f0d4d963830e1cb99ad",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "039f7eeba09d957551ec687e23cd7c5c6c3c8813",
              "role": "Raadslid"
            },
            {
              "id": "c04c0a57fd78cbfb2bd225aed7e12a8c",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "95a1340b1f85c9cf321d8b457e43d0b4e3c02630",
              "role": "Fractievoorzitter"
            },
            {
              "id": "2e74c37401f62b822c72c3208da5f10e",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "d4d2842da0431de7fcf96d6bff70ca2ab57a6d5f",
              "role": "Fractievoorzitter"
            },
            {
              "id": "d387e345978c9c56e7e04a1ecc3c2f0d",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "d646ba9bd45cbb9fc2d221eb742c33d95b426e37",
              "role": "Fractievoorzitter"
            },
            {
              "id": "127a64b51da7832890b0ec19bf5d1785",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "a410897b128cfed86b3c8f1d3672199995d7c308",
              "role": "Fractievoorzitter"
            },
            {
              "id": "03fb396d173032d910856bb239478ecb",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "02d155142173732db9844c923bc21fa4c2027617",
              "role": "Raadslid"
            },
            {
              "id": "d92afc688cc98881c496a2b78fb6cfaa",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "6e272921008f8aad0b26f0088857387167906237",
              "role": "Raadslid"
            },
            {
              "id": "eba28f9038c03dce0d9c4703c1a627c9",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "559a7ca37aa5db8c4abea33085a471f064da5df9",
              "role": "Raadslid"
            },
            {
              "id": "7067d473e6d6846486f71442c997e5c8",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "d278efe32f8d9c9af8756e11e452769a0ba313da",
              "role": "Raadslid"
            },
            {
              "id": "b725f9a5add915db7bd3ddaa6a673cda",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "2733142a65980a4a0b132c4750204517e8f44fd7",
              "role": "Fractievoorzitter"
            },
            {
              "id": "8e65b86c369165c5f7b5f8c6295b6300",
              "label": "Burgemeester",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "8e5dd772af1e10094f752af0b2603d7a154567cc",
              "role": "Burgemeester"
            },
            {
              "id": "7e22bf801394e44955b43001490e11c4",
              "label": "Wethouder",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "2b5a390f983a399c125c156941f5e4725ede7b90",
              "role": "Wethouder"
            },
            {
              "id": "31b21ceb295764eb17ec9b8ec681408f",
              "label": "Locoburgemeester",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "d0c550bfc092b328b223550a01340732e420e511",
              "role": "Locoburgemeester"
            },
            {
              "id": "b52ca6b157cec72e5b7164f01c0ec1a6",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "821d9d71513879cb284709e9584ada6909ece037",
              "role": "Raadslid"
            },
            {
              "id": "db6ba1f6475fe2256de78a3a147c6d6c",
              "label": "Wethouder",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "ca50997d99aaae6a783e1303b8d4893cc873fa95",
              "role": "Wethouder"
            },
            {
              "id": "a8f997d6e261ba78bb83e1c721503ef0",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "c19d4b7a5046aa6e73fd227bd6e3c6f1b66aa44b",
              "role": "Fractievoorzitter"
            },
            {
              "id": "608f93b32e623e15093ed3cb544a7807",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "1186b8ea893bf87492695965af2f83d709605cd6",
              "role": "Raadslid"
            },
            {
              "id": "a66276f9027945f9b0f2e59f819c35c2",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "8fb8741f019c064f17e1ae689d94304f72b7df1d",
              "role": "Raadslid"
            },
            {
              "id": "78d27018efe4863fb5f60178e442c9b4",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "a3138426114d33d33437b6d08953f2219fdc2f61",
              "role": "Raadslid"
            },
            {
              "id": "34d2dd8af2b5eba0f626e06390148c70",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "cfbfa74d0a3f287c6bb2bc4c69e5593bf9c524d9",
              "role": "Raadslid"
            },
            {
              "id": "e4c1d40090fcffc8563558ace7058219",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "7c1196b22f51a4d2209b3250b983cc5e3a51e00d",
              "role": "Fractievoorzitter"
            },
            {
              "id": "011bfe6bc27a38c3416815f5c8e3198c",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "95dda1ebd1f73c2e6e1690767964c43afba2ff8f",
              "role": "Fractievoorzitter"
            },
            {
              "id": "b9d932d0eeab0d7a840003ac3b2c5a8f",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "cccf0890c3f50ddd3bea5e3fce00b77c5e93bf51",
              "role": "Fractievoorzitter"
            },
            {
              "id": "5b53b0f9770cc37f4934516f6460964c",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "a5db41e17594e64bfe585fa15875fdb41f743418",
              "role": "Raadslid"
            },
            {
              "id": "2522b3ee5447b5560747a67618226360",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "e20680e9a2482a0448ef2188234fe2c85290c14b",
              "role": "Raadslid"
            },
            {
              "id": "166303bda5a9939e74ff2882ff98bcd6",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "2e74af7852253e2b5a3d5bb314b2105bd69145e7",
              "role": "Raadslid"
            },
            {
              "id": "361e428def7493e95703178a437ff7e2",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "e8dcdb5493728496aa50ad9bb97d78fbd1384c48",
              "role": "Raadslid"
            },
            {
              "id": "f310fc8d583de9213b4f0f8dad8e3a13",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "d3636c5af5094f9bcd88376f7233ae77c1947ddd",
              "role": "Raadslid"
            },
            {
              "id": "871d641b4bb0871f07f3ec13db65426e",
              "label": "Fractievoorzitter",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "16fb10cd464932a7b30018a51238e4241dcca617",
              "role": "Fractievoorzitter"
            },
            {
              "id": "a14534228eabed8317c7e24df57595fa",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "f9bb05475fcb46d28dd32510fdf528d7e4095e09",
              "role": "Raadslid"
            },
            {
              "id": "70e03d44e2843cb8b78704ac900b037a",
              "label": "Raadslid",
              "organization": {
                "classification": "Council",
                "id": "gemeente-den-helder-den-helder-nh",
                "name": "Gemeente Den Helder (Den Helder) (NH)"
              },
              "organization_id": "gemeente-den-helder-den-helder-nh",
              "person_id": "8e11e8fd7d38c7fc576290209f6e11c3a4f80d31",
              "role": "Raadslid"
            }
          ],
          "meta": {
            "_score": 1.0,
            "_type": "organizations",
            "collection": "Gemeente Den Helder (Den Helder) (NH)",
            "ocd_url": "http://127.0.0.1:5000/v0/den_helder_popit_organizations/gemeente-den-helder-den-helder-nh",
            "original_object_id": "gemeente-den-helder-den-helder-nh",
            "original_object_urls": {
              "html": "https://almanak.overheid.nl/24611/Gemeente_Den_Helder/"
            },
            "processing_finished": "2015-10-12T15:05:23.665996",
            "processing_started": "2015-10-12T15:05:23.617484",
            "rights": "undefined",
            "source_id": "den_helder_popit_organizations"
          },
          "name": "Gemeente Den Helder (Den Helder) (NH)"
        },
        "organization_id": "gemeente-den-helder-den-helder-nh",
        "sources": [
          {
            "note": "",
            "url": "https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/"
          },
          {
            "description": "gemeente \nDen Helder \n\nde  R a ad  v an  de  g e m e e n te  D en  H e l d er \nP o s t b us  36 \n1 7 80  AA  D EN  H E L D ER \n\n: 7 oktober 2015 \n\nverzendgegevens \ndatum \nkenmerk  : AU15.10692 \nbijlagen \n: \nonderwerp \nOproep vergadering  gemeenteraad \n\nG e a c h te  r a a d, \n\nbehandeld  door \nGriffie \ndhr.  F. Blok \ntelefoon  (0223) 67  8103 \n\nuw gegevens \nbrief van  : \nkenmerk  : \n\nHierbij  n o d ig  ik  u uit  tot  het  b i j w o n en  v an  de  r a a d s v e r g a d e r i ng  op  maandag 12 oktober 2015  om \n1 9 : 30  u ur  in  de  r a a d z a al  v an  h et  s t a d h u is  v an  D en  H e l d e r.  B i j g e v o e gd  treft  u de  a g e n da  a a n. \nDe  op  de  a g e n da  b e t r e k k i ng  h e b b e n de  s t u k k e n,  v o or  z o v er  d e ze  niet  zijn  m e e g e z o n d en  of  r e e ds  in  uw \nbezit  zijn,  zijn  v o or  u te  r a a d p l e g en  op  h t t p : / / g e m e e n t e r a a d . d e n h e l d e r . nl  en  liggen  o ok  v o or  u ter  i n z a ge in \nde  l e e s k a m er  v an  7 o k t o b er  2 0 15  ( v a n af  1 6 . 00  u u r)  tot  en  m et  12  o k t o b er  2 0 15  (tot  1 6 . 00  u u r ). \n\nI n d i en  u m o t i es  en  a m e n d e m e n t en  hebt,  die  u v o o r af  b e k e nd  w e n st  te  m a k e n,  d an  k u nt  u d e ze  v\u00f3\u00f3r  1 3 . 00 \nu ur  op  de  d ag  v an  de  r a a d s v e r g a d e r i ng  bij  de  raadsgriff\u00ece  k e n b a ar  m a k e n.  De  griffie  z al  d e ze  m o t i es  en \na m e n d e m e n t en  per  o m g a a nd  d o o r m a i l en  n a ar  alle  r a a d s l e d e n. \n\nV r a g en  v o or  het  v r a g e n k w a r t i er  d i e nt  u uiterlijk  24  u ur  v o or  de  a a n v a ng  v an  de  v e r g a d e r i ng  a an  te  m e l d en \nbij  de  raadsgriff\u00ece. \n\nV o or  alle  z a k en  r o nd  de  r a a d s v e r g a d e r i ng  k u nt  u het  v o l g e n de  e - m a i l a d r es  h a n t e r e n: \ng r i f f i e @ d e n h e l d e r . n l. \n\nH o o g a c h t e n d, \nde  v o o r z i t t er  v an  de  g e m e e n t e r a a d, \nn a m e ns  de \n\nmr.  drs \n\nu  s m an \n\nDrs. F. Bijlweg  20 \n1784  MC  D en  Helder \n\nPostbus  36 \n1780 AA  Den  Helder \n\nwww.denhelder.nl \nkcc@denhelder.nl \n\ntelefoon 14  0223 \nfax (0223) 67  1201 \n\n\f",
            "note": "Oproep vergadering gemeenteraad.pdf",
            "url": "https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/Oproep-vergadering-gemeenteraad-9.pdf"
          }
        ],
        "start_date": "2015-10-12T19:30:00",
        "status": "confirmed"
      }



.. http:get:: /(source_id)/(doc_type)/(object_id)/source

   Retrieves the object's data in its original and unmodified form, as supplied by the data source. Being able to retrieve the object in it's original form can be useful for debugging purposes (i.e. when fields are missing or odd values are returned in the representation of the object).

   The value of the ``Content-Type`` response header depends on the type of data that is returned by the data provider.

   :statuscode 200: OK, no errors.
   :statuscode 404: The requested source and/or object does not exist.

   **Example request**

   .. sourcecode:: http

      $ curl -i 'http://api.openraadsinformatie.nl/v0/den_helder/events/4a39c497c7818af9ad6d7d8335a6e951d6a83bd6/source'

   **Example response**

   .. sourcecode:: http

      HTTP/1.1 200 OK
      content-type: application/json
      content-length: 1339
      date: Wed, 27 May 2015 12:57:47 GMT

      {"content": "<html xmlns=\"http://www.w3.org/1999/xhtml\" xml:lang=\"nl\" lang=\"nl\">&#13;\n\t<head>&#13;\n\t\t<meta name=\"description\" content=\"-\"/>&#13;\n\t\t<title>Gemeenteraad 12 oktober 2015 19:30:00</title>&#13;\n\t\t<meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/>&#13;\n\t\t<meta name=\"author\" content=\"GemeenteOplossingen - http://www.gemeenteoplossingen.nl\"/>&#13;\n\t\t<meta name=\"generator\" content=\"GO. raadsinformatie 5.4\"/>&#13;\n        <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\"/>&#13;\n&#13;\n        <meta property=\"og:type\" content=\"website\"/>&#13;\n        <meta property=\"og:title\" content=\"Vergadering\"/>&#13;\n        <meta property=\"og:url\" content=\"https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"/>&#13;\n        <meta property=\"og:image\" content=\"https://gemeenteraad.denhelder.nl/site/den-helder2015/images/home_visual.jpg\"/>&#13;\n&#13;\n\t<meta name=\"google-site-verification\" content=\"\"/>&#13;\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/default.css\" media=\"all\"/>&#13;\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/layout.css\" media=\"all\"/>&#13;\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/jquery-ui-1.8.18.custom.css\" media=\"all\"/>&#13;\n\t\t&#13;\n\t&#13;\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/meeting.css\" media=\"all\"/>&#13;\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/den-helder2015/css/client.css\" media=\"all\"/>\n&#13;\n&#13;\n\t\t&#13;\n\t\t<link rel=\"home\" title=\"Home\" href=\"/\"/>&#13;\n\t\t<link rel=\"contents\" title=\"Sitemap\" href=\"/sitemap\"/>&#13;\n\t\t<link rel=\"search\" title=\"Zoeken\" href=\"/zoeken\"/>&#13;\n&#13;\n\t<!--\r\n\t\tTechnische realisatie:\t\r\n\t\t\r\n\t\tGemeenteOplossingen\r\n\t\thttp://www.gemeenteoplossingen.nl\r\n\t-->&#13;\n&#13;\n\t<!-- vergadering -->&#13;\n\t</head>&#13;\n&#13;\n\t<body id=\"top\">&#13;\n\t\t<div id=\"page\" class=\"vergadering\">&#13;\n\t\t\t&#13;\n\t\t\t<!-- Main container, content wrapper -->&#13;\n\t\t\t<div id=\"wrapper\" class=\"container\">&#13;\n\t\t\t\t&#13;\n\t\t\t\t<h1 class=\"nonvisual\">Gemeenteraad 12 oktober 2015 19:30:00</h1>&#13;\n\t\t\t\t<a class=\"skip\" href=\"#header\">Content overslaan</a>&#13;\n\t\t\t\t&#13;\n\t\t\t\t<div id=\"breadcrumb\">U bent hier: <a href=\"/vergaderingen/\">Vergaderingen</a> <span class=\"divider\">&#187;</span> <a href=\"/vergaderingen/Gemeenteraad/\">Gemeenteraad</a> <span class=\"divider\">&#187;</span> <a href=\"/vergaderingen/Gemeenteraad/2015/\">2015</a> <span class=\"divider\">&#187;</span> <a href=\"/vergaderingen/Gemeenteraad/2015/12-oktober/\">12-oktober</a> <span class=\"divider\">&#187;</span> <strong>19:30</strong></div>&#13;\n\t\t\t\t &#13;\n\t\t\t\t<div id=\"content\">&#13;\n\t\t\t\t\t<input type=\"hidden\" value=\"17673\" id=\"meeting_object_id\"/>&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"uitzending_meeting\" class=\"broadcast\">&#13;\n\t\t\t\t\t\t    <script type=\"text/javascript\" src=\"/jwplayer/jwplayer.js\"/>\n    <script type=\"text/javascript\">jwplayer.key=\"UCSfwBt+qmRTLBgiLL70hAT6TP3dtA29Yg7HEw==\";</script>\n    <div id=\"live_player\"><div id=\"mediaplayer_live\"/></div>\n&#13;\n\t\t\t\t\t\t\t<div id=\"archive_player_android_container\"><div id=\"archive_player_android\"/><div id=\"android_links\"/></div>\n&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t<input type=\"hidden\" id=\"currentObjectIds\" value=\"17673\"/>\n&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t</div>&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"alternate_live_streams\">&#13;\n\t\t\t\t\t\t<h2>Kijk of luister mee </h2>&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t<div class=\"clearer\">&#160;</div>&#13;\n\t\t\t\t\t</div>&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"pageHead\">&#13;\n\t\t\t\t\t\t<div class=\"meta\">&#13;\n\t\t\t\t\t\t\t<h2 class=\"page_title\"><span class=\"highlighted group\">Gemeenteraad</span> <span class=\"date\">12 oktober 2015</span> <span class=\"time\">19:30</span> <span class=\"hour\">uur</span> </h2>&#13;\n                            &#13;\n                            &#13;\n\t\t\t\t\t\t\t<p class=\"meeting_extra\"/>&#13;\n\t\t\t\t\t\t</div>&#13;\n\t\t\t\t\t\t<div class=\"actions\">&#13;\n\t\t\t\t\t\t\t<a class=\"print_pagina\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/print\">Print deze agenda</a>\n\t\t\t\t&#13;\n\t\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t\t<a class=\"print_pagina download_documenten\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/alle-documenten\">Download alle documenten</a>\n&#13;\n\t\t\t\t\t\t\t<div id=\"title_note_icon\"/>&#13;\n                            <div id=\"document_legend\" class=\"active\"><h3>Legenda</h3>\n<ul>\t<li class=\"aangeboden\">Aangeboden</li>\t\t<li class=\"agenda\">Agenda</li>\t\t<li class=\"amendement\">Amendement</li>\t\t<li class=\"beantwoording-raadsvragen\">Beantwoording raadsvragen</li>\t\t<li class=\"brief\">Brief</li>\t\t<li class=\"brieven-van-buiten\">Brieven van buiten</li>\t\t<li class=\"documentsoorten\">Documentsoorten</li>\t\t<li class=\"informatie\">Informatie</li>\t\t<li class=\"ingekomen-brieven\">ingekomen brieven</li>\t\t<li class=\"ingekomen-stuk\">Ingekomen stuk</li>\t\t<li class=\"ingekomen-stukken\">Ingekomen stukken</li>\t\t<li class=\"inhoud-map-losse-stukken\">Inhoud map losse stukken</li>\t\t<li class=\"kennisgeving\">Kennisgeving</li>\t\t<li class=\"lange-termijn-agenda\">Lange termijn agenda</li>\t\t<li class=\"mededelingen\">Mededelingen</li>\t\t<li class=\"memo\">Memo</li>\t\t<li class=\"notitie\">Motie</li>\t\t<li class=\"nieuwsbrief\">Nieuwsbrief</li>\t\t<li class=\"nieuwsbrieven\">Nieuwsbrieven</li>\t\t<li class=\"nota\">Nota</li>\t\t<li class=\"onbekend\">Onbekend</li>\t\t<li class=\"openbare-besluitenlijst-bw\">Openbare besluitenlijst B&amp;W</li>\t\t<li class=\"overig\">Overig</li>\t\t<li class=\"overige-bijlagen\">Overige bijlagen</li>\t\t<li class=\"overzicht-toezeggingen\">Overzicht toezeggingen</li>\t\t<li class=\"raadsbesluit\">Raadsbesluit</li>\t\t<li class=\"raadsinformatiebrief\">Raadsinformatiebrief</li>\t\t<li class=\"raadvoorstel\">Raadsvoorstel</li>\t\t<li class=\"speciale documenten\">Speciale Documenten</li>\t\t<li class=\"stukken-ter-inzage\">Stukken ter inzage</li>\t\t<li class=\"stukken-ter-kennisname\">Stukken ter kennisname</li>\t\t<li class=\"tvb\">TVB</li>\t\t<li class=\"uitnodiging\">Uitnodiging</li>\t\t<li class=\"uitnodigingen\">Uitnodigingen</li>\t\t<li class=\"vergaderschema\">Vergaderschema</li>\t\t<li class=\"verordening\">verordening</li>\t\t<li class=\"verslag\">Verslag</li>\t\t<li class=\"vragen-van-raadsleden\">Vragen van raadsleden</li>\t\t<li class=\"wandelgang\">Wandelgang</li>\t\t<li class=\"vertrouwlijk\">Vertrouwelijk document</li>\t\n</ul>\n\n</div>&#13;\n\t\t\t\t\t\t</div>&#13;\n\t\t\t\t\t\t&#13;\n\t\t\t\t\t\t<span class=\"clearer\">&#160;</span>&#13;\n\t\t\t\t\t</div>&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"documenten\">\n<h2>Vergaderstukken:</h2>\n<ul>\n\t<li class=\"onbekend\">\n\t<a href=\"Oproep-vergadering-gemeenteraad-9.pdf\">Oproep vergadering gemeenteraad.pdf</a>\n\t<span class=\"extention pdf\">(pdf, 486.13 kb)</span>\n\t<a href=\"Oproep-vergadering-gemeenteraad-9.pdf/notitie/\" title=\"Notitie\" class=\"notelink \" id=\"notelink17667_0_\">\n\t\t<span class=\"nonvisual\">notitie</span>\n\t</a>\n</li>\n</ul>\n</div>\n&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"agendapunten\">&#13;\n\t\t\t\t\t\t<h2>Agenda</h2>\n<ul id=\"vergadering\" class=\"hilite\">\n<li id=\"agendapunt17672_0\" class=\"actief agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\">1</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\"> Opening.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17672_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17672\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_17672\" class=\"attachementRow attachement\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17672\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17672\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17672\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_17672\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17672\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17672\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_17672\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17672_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19254_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">2</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Verslag commissie onderzoek geloofsbrieven en be&#235;diging en installatie van de benoemde raadsleden, mevrouw A. Hogendoorn en de heren R. van Deutekom, A.A.H. Koenen en A.J. Pruiksma.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19254_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19254\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19254\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19254\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19254\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19254\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19254\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19254\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19254\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19254\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19254_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17671_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\">3</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\"> Spreekrecht burgers.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17671_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17671\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_17671\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17671\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17671\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17671\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_17671\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17671\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17671\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_17671\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17671_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17670_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\">4</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\"> Vragenkwartier.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17670_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17670\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_17670\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17670\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17670\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17670\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_17670\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17670\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17670\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_17670\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17670_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17669_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\">5</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\"> Bepalen stemvolgorde.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17669_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17669\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_17669\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17669\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17669\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17669\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_17669\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17669\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17669\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_17669\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17669_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17668_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\">6</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\"> Vaststelling agenda.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17668_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button17668\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_17668\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17668\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17668\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17668\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_17668\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17668\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17668\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_17668\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17668_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19198_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">7</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het wijzigen van de Algemene plaatselijke verordening 2012 voor het gebruik van knalapparatuur in de landbouw.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Voorgesteld wordt het in de APV gebezigde ontheffingsstelsel voor het gebruik van knalapparatuur voor het verjagen van vogels en wild in de landbouw te vervangen door een stelsel met algemene regels.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19198_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19198\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19198\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19198\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19198\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19198\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19198\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19198\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19198\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19198\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19198_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19245_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">8</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel met betrekking tot de wijziging van de Gemeenschappelijke Regeling Regionale Uitvoeringsdienst Noord-Holland Noord (RUD NHN).</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">De RUD NHN voert sinds 1 januari 2014 voornamelijk milieu- en bodemtaken uit voor de provincie en de gemeenten op basis van de Gemeenschappelijke Regeling Noord-Holland Noord. Naar aanleiding van een wijziging van de Wet gemeenschappelijke regelingen en wegens het feit dat nog een aantal punten in de regeling nader dienden te worden ingevuld, wordt de raad voorgesteld in te stemmen met een aantal aanpassingen van de gemeenschappelijke regeling.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19245_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19245\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19245\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19245\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19245\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19245\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19245\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19245\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19245\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19245\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19245_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19199_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\">9</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\"> Voorstel tot het vaststellen van de Kwijtscheldingsverordening 2015.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Per 1 januari 2015 is de uitvoering van de gemeentelijke belastingen overgegaan naar de Gemeenschappelijke Regeling Cosensus. Omdat Cosensus ook een deel van de invordering uitvoert waaronder de kwijtschelding van de gemeenteljike belastingen dient ook de kwijtscheldingsverordening geactualiseerd en opnieuw vastgesteld te worden.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19199_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19199\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19199\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19199\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19199\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19199\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19199\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19199\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19199\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19199\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19199_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19233_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">10</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van de Welstandsnota 2015 en de voorgestelde wijziging van de reclamenota \"Bedrijven (positief) in beeld\".</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Op 17 maart 2014 heeft de raad de kaderstelling voor de herziening van de Welstandsnota vastgesteld. Thans wordt de raad voorgesteld de op deze kaderstelling hierziene nota, de Welstandsnota 2015, vast te stellen. De nota is helder en compact geworden met het accent op behoud en stimuleren van de ruimtelijke kwaliteit.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19233_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19233\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19233\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19233\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19233\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19233\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19233\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19233\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19233\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19233\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19233_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19253_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">11</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van de Hoofdlijnen van aanpak voor een duurzaam en klimaatbestendig Den Helder.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">De gemeente speelt een belangrijke rol bij het communiceren, stimuleren en faciliteren van initiatieven op het gebied van duurzaamheid en klimaatbestendigheid. De komende vier jaar worden alleen die zaken opgepakt waarop binnen deze termijn vooruitgang te maken is.  Het betreft duurzaam handelen bij ruimtelijke ontwikkelingen, energiebesparingsmaatregelen in de bestaande woningvoorraad en bij bedrijven, bodemsaneringen, afvalrecycling, duurzame economie stimuleren, het goede voorbeeld geven als gemeente, stimulering plaatselijke energieproductie, bewonersinitiatieven en bewustwordingsacties.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19253_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19253\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19253\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19253\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19253\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19253\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19253\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19253\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19253\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19253\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19253_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19247_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">12</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van het bestemmingsplan Huisduinen en de Stelling 2015.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Het bestemmingsplan voorziet in een actuele regeling voor het plangebied, waarin alle beleidsstukken zijn meegenomen. Tot het plangebied behoren ook de Stelling met de forten Erfprins en Dirksz Admiraal en de daarbij behorende voormalige schootsvelden.  In het bestemmingsplan wordt ook geanticipeerd op de wensen van de Huisduiner Ontwikkelings Maatschappij voor de ontwikkeling van het gebied Huisduinerkwartier. Er zijn acht zienswijzen op het ontwerp-bestemmingsplan ontvangen. Het voorstel is om deze zienswijzen gedeeltelijk te volgen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19247_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19247\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19247\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19247\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19247\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19247\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19247\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19247\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19247\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19247\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19247_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19246_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">13</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel met betrekking tot de tussenrapportage 2015.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Als onderdeel van de planning- en controlcyclus biedt het college van burgemeester en wethouders de tussenrapportage 2015 aan. De rapportage is gebaseerd op de periode januari tot en met augustus en geeft de voortgang en afwijkingen aan ten opzichte van de programmabegroting 2015. De raad dient de bijbehorende wijziging van de programmabegroting vast te stellen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19246_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19246\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19246\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19246\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19246\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19246\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19246\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19246\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19246\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19246\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19246_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19248_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">14</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van het Integraal Huisvestingsplan 2015-2030 met het daarbij behorende investeringsoverszicht en aanbevelingen.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Het Integraal Huisvestingsplan 2015-2030 laat zien hoe de gemeente de komende periode wil omgaan met de onderwijshuisvesting op het grondgebied van de gemeente. In het Integraal Huisvestingsplan wordt met name ingezet op de koers die met schoolbesturen is ingezet in het kader van de daling van het aantal leerlingen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19248_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19248\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19248\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19248\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19248\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19248\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19248\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19248\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19248\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19248\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19248_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19250_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\">15</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\"> Voorstel tot het voorzien in de gemeentelijke vertegenwoordiging in de besturen van de zes gemeenschappelijke regelingen.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19250_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19250\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19250\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19250\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19250\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19250\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19250\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19250\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19250\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19250\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19250_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19251_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">16</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot benoeming van leden van de Noordkopraad.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19251_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19251\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19251\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19251\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19251\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19251\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19251\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19251\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19251\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19251\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19251_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19249_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">17</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het benoemen van leden in de auditcommissie.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19249_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19249\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19249\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19249\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19249\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19249\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19249\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19249\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19249\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19249\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19249_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19252_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">18</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot benoeming van een lid van de vertrouwenscommissie.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19252_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19252\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19252\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19252\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19252\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19252\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19252\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19252\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19252\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19252\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19252_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19234_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">19</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel voor herstel en herbestemming van het Logementsgebouw (het Casino) in Huisduinen.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">Het Logementsgebouw  is een zeer bijzonder gebouw binnen het oorlogserfgoed van Den Helder, maar ook binnen het oorlogserfgoed van Nederland. Het logementsgebouw vormt samen met de loods en het poortgebouw een monumentaal complex dat op de lijst van Rijksmonumenten is geplaatst. Het voornemen is het gebouw te ontwikkelen als Atlantikwall Expeditie Centrum, een bezoekerscentrum met een interactieve presentatie. Uitvalsbasis voor een bezoek aan de Stelling van Den Helder en de Atlantikwall in Den Helder en het Waddengebied. Voorgesteld wordt (financi&#235;le) ondersteuning te bieden, door cofinanciering van Waddenfondssubsidie.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19234_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19234\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19234\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19234\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19234\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19234\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19234\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19234\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19234\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19234\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19234_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19235_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">20</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het beschikbaar stellen van een krediet van &#8364; 4,7 mln. voor de verwerving van de benodigde gronden en de realisatie van de Noorderhaaks, inclusief deelprojecten.</a></h3>\n\t\t<em/>\n\t\t<div class=\"toelichting\">De raad heeft het college opgedragen een voorstel aan de raad voor te leggen voor de aanleg van de Noorderhaaks. Het college stelt thans de raad voor een krediet van &#8364; 4,7 mln beschikbaar te stellen. De investering wordt deels bekostigd door een provinciale subsidie en deels vanuit gemeentelijke reserveringen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19235_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19235\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_19235\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19235\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19235\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19235\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_19235\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19235\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19235\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_19235\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19235_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17667_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\">21</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\"> Sluiting.</a></h3>\n\t\t<em/>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17667_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17667\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"/>\n\n\t<div id=\"attachements_17667\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17667\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17667\" class=\"fragments\"/>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17667\" class=\"speaking\"/>\n\t\t\t\n\t\t\t<div id=\"persons_17667\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17667\" class=\"speakers\"/>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17667\" class=\"more\"/>\n\t\t\t<div id=\"agendapunt_documenten_17667\" class=\"documents\"/>\n\t\t</div>\n\t\t<div class=\"clearer\"/>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17667_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n\n</ul><!--eof vergadering-->\n<input type=\"hidden\" name=\"fragment\" id=\"fragment\" value=\"0\"/>\n&#13;\n\t\t\t\t\t</div>&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"status_uitzending\"/>&#13;\n\t\t\t\t\t<div id=\"presentation\" class=\"leeg\">&#13;\n\t\t\t\t\t\t<img id=\"vergroten\" src=\"/md/zoomDoc.gif\" alt=\"afbeelding vergroten\" onclick=\"togglesize('#presentation_sheet')\"/>&#13;\n                        <img id=\"presentation_sheet\" src=\"/md/empty.jpg\" title=\"klik om te vergroten/verkleinen\" alt=\"laden...\" class=\"normal_sheet\" onclick=\"togglesize(this)\"/>&#13;\n\t\t\t\t\t</div>&#13;\n\t\t\t\t</div>&#13;\n\t\t\t\t&#13;\n\t\t\t\t<div class=\"clearer\"> </div>&#13;\n\t\t\t</div>&#13;\n\t\t\t&#13;\n\t\t\t<!-- Header -->&#13;\n\t\t\t<div id=\"header\">&#13;\n\t\t\t\t<div class=\"container\">&#13;\n\t\t\t\t\t<div id=\"logo\"><a href=\"/\" title=\"Terug naar home\">Gemeente Den Helder</a></div>&#13;\n\t\t\t\t\t&#13;\n\t\t\t\t\t<div id=\"options\"><div class=\"options_wrapper\">&#13;\n\t<h3 class=\"nonvisual\">Pagina opties</h3>&#13;\n\t<ul>&#13;\n\t\t<li><a class=\"rss\" href=\"/rss_vergaderingen\">RSS</a></li>&#13;\n\t\t<li><a class=\"print\" href=\"/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/print\">Afdrukken</a></li>&#13;\n\t\t<li><a class=\"readspeak\" href=\"/\">Lees voor</a></li>&#13;\n\t\t<li>&#13;\n\t\t\t<a class=\"text_sml\" href=\"/zoom/level1\" title=\"Kleinere letters\"><span class=\"nonvisual\">Kleinere letters</span></a>&#13;\n\t\t\t<a class=\"text_mid\" href=\"/zoom/level2\" title=\"Normale letters\"><span class=\"nonvisual\">Normale letters</span></a>&#13;\n\t\t\t<a class=\"text_lrg\" href=\"/zoom/level3\" title=\"Grote letters\"><span class=\"nonvisual\">Grote letters</span></a>&#13;\n\t\t</li>&#13;\n\t</ul>&#13;\n</div></div>&#13;\n\t\t&#13;\n\t\t\t\t\t<div id=\"search\"><h3 class=\"nonvisual\" id=\"zoeken\">Zoeken</h3>&#13;\n<form action=\"/zoek/\" method=\"post\">&#13;\n\t<fieldset>&#13;\n\t\t<legend>Snelzoeken</legend>&#13;\n\t\t<label class=\"nonvisual\" for=\"q\">Zoekterm</label>&#13;\n\t\t<input type=\"text\" id=\"q\" name=\"zoek_opdracht\" value=\"Zoeken...\"/>&#13;\n\t\t<input type=\"submit\" id=\"search_button\" name=\"vind\" value=\"Zoek\"/>&#13;\n\t\t<input type=\"hidden\" value=\"alle_woorden\" class=\"hidden\" name=\"search\"/>&#13;\n\t</fieldset>&#13;\n</form>&#13;\n</div>&#13;\n\t\t\t\t</div>&#13;\n\t\t\t\t&#13;\n\t\t\t\t<div id=\"menuHolder\">&#13;\n\t\t\t\t\t<div class=\"container\">&#13;\n\t\t\t\t\t\t<h2 class=\"nonvisual\">Site Navigatie</h2>&#13;\n\t\t\t\t\t\t<a class=\"skip\" href=\"#header\">Site Navigatie overslaan</a>&#13;\n\t\t\t\t\t\t<h3 class=\"nonvisual\">Hoofdmenu</h3>\n<ul id=\"menu\">\n<li><a href=\"/\"><span>raad home</span></a></li>\n<li><a href=\"/Actueel\"><span>actueel</span></a></li>\n<li><a href=\"/Organisatie\"><span>organisatie</span></a></li>\n<li><a class=\"active\" href=\"/Vergaderingen\"><span>vergaderingen</span></a></li>\n<li><a href=\"/Documenten\"><span>documenten</span></a></li>\n<li><a href=\"/Abonnement\"><span>abonnement</span></a></li>\n<li><a href=\"/Kalender\"><span>kalender</span></a></li>\n</ul>\t&#13;\n\t\t\t\t\t</div>&#13;\n\t\t\t\t</div>&#13;\n\t\t\t\t&#13;\n\t\t\t\t<div id=\"user\">&#13;\n\t\t\t\t\t<div class=\"container\"> <h3 class=\"nonvisual\">Inloggen</h3>\n<p class=\"nonvisual\">U kunt inloggen als raadslid: </p><a id=\"inloggen_link\" href=\"/inloggen?url=/vergaderingen/Gemeenteraad/2015/12-oktober/19:30\">Inloggen</a>\n </div>&#13;\n\t\t\t\t</div>&#13;\n\t\t\t</div>&#13;\n\t\t\t&#13;\n\t\t</div>&#13;\n        <!-- Footer -->&#13;\n        <div id=\"footer\">&#13;\n            <div class=\"container\">&#13;\n\t<div class=\"block\">&#13;\n\t\t<h3>Contact</h3>&#13;\n\t\t<address>&#13;\n\t\t\t<p>&#13;\n\t\t\t\t<span>Raadsgriffie gemeente Den Helder</span>&#13;\n\t\t\t\t<span>Drs. F. Bijlweg 20</span>&#13;\n\t\t\t\t<span>Postbus 36</span>&#13;\n\t\t\t\t<span>1780 AA, Den Helder</span>&#13;\n\t\t\t</p>&#13;\n\t\t\t&#13;\n\t\t\t<p>&#13;\n\t\t\t\t<span>T: 14 0223</span>&#13;\n\t\t\t\t<span>F: (0223) 67 12 01</span>&#13;\n\t\t\t\t<span><a href=\"mailto:griffie@denhelder.nl\" title=\"Mail naar gemeente Den Helder\">griffie@denhelder.nl</a></span>&#13;\n\t\t\t</p>\t\t&#13;\n\t\t</address>&#13;\n\t</div>&#13;\n\t&#13;\n\t<div class=\"block\">&#13;\n\t\t<h3>Over deze site</h3>&#13;\n\t\t<ul>&#13;\n\t\t\t<li><a class=\"home\" href=\"/\"><span>Home</span></a></li>&#13;\n\t\t\t<li><a class=\"contact\" href=\"/Over-deze-site/Contact/\">Contact</a></li>&#13;\n\t\t\t<li><a class=\"disclaimer\" href=\"/Over-deze-site/Disclaimer/\">Disclaimer</a></li>&#13;\n\t\t\t<li><a class=\"colofon\" href=\"/Over-deze-site/Colofon/\">Colofon</a></li>\t&#13;\n\t\t\t<li><a class=\"toegankelijkheid\" href=\"/Over-deze-site/Toegankelijkheid/\">Toegankelijkheid</a></li>\t&#13;\n\t\t\t<li><a class=\"sitemap\" href=\"/Over-deze-site/Sitemap/\">Sitemap</a></li>\t&#13;\n\t\t\t<li><a class=\"zoeken\" href=\"/zoeken\">Zoeken</a></li>\t&#13;\n\t\t</ul>&#13;\n\t</div>&#13;\n\t&#13;\n\t<div class=\"block\">&#13;\n\t\t<h3>Volg ons op</h3>&#13;\n\t\t<ul>&#13;\n\t\t\t<li><a class=\"rss\" href=\"/rss\"><span>RSS</span></a></li>&#13;\n\t\t</ul>&#13;\n\t\t<br/>&#13;\n\t\t<h3>Gemeentelijke website</h3>&#13;\n\t\t<ul>&#13;\n\t\t\t<li><a href=\"http://www.denhelder.nl\">Gemeente Den Helder</a></li>&#13;\n\t\t</ul>&#13;\n\t</div>&#13;\n\t&#13;\n\t<a href=\"#top\" id=\"toTop\">Terug naar boven</a>&#13;\n\t&#13;\n</div>&#13;\n<div id=\"copy\">&#13;\n\t<p class=\"container\">&#13;\n\t\t<span>&#169;2015 Gemeente Den Helder</span>&#13;\n\t\t<a href=\"https://www.gemeenteoplossingen.nl/producten/papierloos_werken/go_raadsinformatie/\" title=\"Meer weten over deze dienst?\" class=\"right\">GemeenteOplossingen | Raadsinformatie</a>&#13;\n\t</p>&#13;\n</div>&#13;\n        </div>&#13;\n        \t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery.js\"/>&#13;\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery-ui.js\"/>&#13;\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery.cookie.js\"/>&#13;\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery.ui.datepicker-nl.js\"/>&#13;\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/activeplx.js\"/>&#13;\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/gui.js\"/>&#13;\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/vergadering.js\"/>&#13;\n&#13;\n        &#13;\n        <script type=\"text/javascript\" src=\"/site/default2014/script/prototype.js\"/>&#13;\n&#13;\n        <script type=\"text/javascript\" src=\"/jwplayer/jwplayer.js\"/>&#13;\n        <script type=\"text/javascript\">jwplayer.key=\"UCSfwBt+qmRTLBgiLL70hAT6TP3dtA29Yg7HEw==\";</script>&#13;\n\t</body>&#13;\n</html>", "full_content": "<!DOCTYPE html>\n<html xmlns=\"http://www.w3.org/1999/xhtml\" xml:lang=\"nl\" lang=\"nl\">\r\n\t<head>\r\n\t\t<meta name=\"description\" content=\"-\" />\r\n\t\t<title>Gemeenteraad 12 oktober 2015 19:30:00</title>\r\n\t\t<meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\" />\r\n\t\t<meta name=\"author\" content=\"GemeenteOplossingen - http://www.gemeenteoplossingen.nl\" />\r\n\t\t<meta name=\"generator\" content=\"GO. raadsinformatie 5.4\" />\r\n        <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\"/>\r\n\r\n        <meta property=\"og:type\" content=\"website\" />\r\n        <meta property=\"og:title\" content=\"Vergadering\" />\r\n        <meta property=\"og:url\" content=\"https://gemeenteraad.denhelder.nl/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" />\r\n        <meta property=\"og:image\" content=\"https://gemeenteraad.denhelder.nl/site/den-helder2015/images/home_visual.jpg\" />\r\n\r\n\t<meta name=\"google-site-verification\" content=\"\" />\r\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/default.css\" media=\"all\"/>\r\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/layout.css\" media=\"all\"/>\r\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/jquery-ui-1.8.18.custom.css\" media=\"all\"/>\r\n\t\t\r\n\t\r\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/default2014/css/meeting.css\" media=\"all\"/>\r\n\t\t<link rel=\"stylesheet\" type=\"text/css\" href=\"/site/den-helder2015/css/client.css\" media=\"all\"/>\n\r\n\r\n\t\t\r\n\t\t<link rel=\"home\" title=\"Home\" href=\"/\" />\r\n\t\t<link rel=\"contents\" title=\"Sitemap\" href=\"/sitemap\" />\r\n\t\t<link rel=\"search\" title=\"Zoeken\" href=\"/zoeken\" />\r\n\r\n\t<!--\r\n\t\tTechnische realisatie:\t\r\n\t\t\r\n\t\tGemeenteOplossingen\r\n\t\thttp://www.gemeenteoplossingen.nl\r\n\t-->\r\n\r\n\t<!-- vergadering -->\r\n\t</head>\r\n\r\n\t<body id=\"top\">\r\n\t\t<div id=\"page\" class=\"vergadering\">\r\n\t\t\t\r\n\t\t\t<!-- Main container, content wrapper -->\r\n\t\t\t<div id=\"wrapper\" class=\"container\">\r\n\t\t\t\t\r\n\t\t\t\t<h1 class=\"nonvisual\">Gemeenteraad 12 oktober 2015 19:30:00</h1>\r\n\t\t\t\t<a class=\"skip\" href=\"#header\">Content overslaan</a>\r\n\t\t\t\t\r\n\t\t\t\t<div id=\"breadcrumb\">U bent hier: <a href=\"/vergaderingen/\" >Vergaderingen</a> <span class=\"divider\">&raquo;</span> <a href=\"/vergaderingen/Gemeenteraad/\" >Gemeenteraad</a> <span class=\"divider\">&raquo;</span> <a href=\"/vergaderingen/Gemeenteraad/2015/\" >2015</a> <span class=\"divider\">&raquo;</span> <a href=\"/vergaderingen/Gemeenteraad/2015/12-oktober/\" >12-oktober</a> <span class=\"divider\">&raquo;</span> <strong>19:30</strong></div>\r\n\t\t\t\t \r\n\t\t\t\t<div id=\"content\">\r\n\t\t\t\t\t<input type=\"hidden\" value=\"17673\" id=\"meeting_object_id\" />\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"uitzending_meeting\" class=\"broadcast\">\r\n\t\t\t\t\t\t    <script type=\"text/javascript\" src=\"/jwplayer/jwplayer.js\"></script>\n    <script type=\"text/javascript\">jwplayer.key=\"UCSfwBt+qmRTLBgiLL70hAT6TP3dtA29Yg7HEw==\";</script>\n    <div id=\"live_player\"><div id=\"mediaplayer_live\"></div></div>\n\r\n\t\t\t\t\t\t\t<div id=\"archive_player_android_container\"><div id=\"archive_player_android\"></div><div id=\"android_links\"></div></div>\n\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t<input type=\"hidden\" id=\"currentObjectIds\" value=\"17673\" />\n\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t</div>\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"alternate_live_streams\">\r\n\t\t\t\t\t\t<h2>Kijk of luister mee </h2>\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t<div class=\"clearer\">&nbsp;</div>\r\n\t\t\t\t\t</div>\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"pageHead\">\r\n\t\t\t\t\t\t<div class=\"meta\">\r\n\t\t\t\t\t\t\t<h2 class=\"page_title\"><span class=\"highlighted group\">Gemeenteraad</span> <span class=\"date\">12 oktober 2015</span> <span class=\"time\">19:30</span> <span class=\"hour\">uur</span> </h2>\r\n                            \r\n                            \r\n\t\t\t\t\t\t\t<p class=\"meeting_extra\"></p>\r\n\t\t\t\t\t\t</div>\r\n\t\t\t\t\t\t<div class=\"actions\">\r\n\t\t\t\t\t\t\t<a class=\"print_pagina\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/print\">Print deze agenda</a>\n\t\t\t\t\r\n\t\t\t\t\t\t\t\r\n\t\t\t\t\t\t\t<a class=\"print_pagina download_documenten\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/alle-documenten\">Download alle documenten</a>\n\r\n\t\t\t\t\t\t\t<div id=\"title_note_icon\"></div>\r\n                            <div id=\"document_legend\" class=\"active\"><h3>Legenda</h3>\n<ul>\t<li class=\"aangeboden\">Aangeboden</li>\t\t<li class=\"agenda\">Agenda</li>\t\t<li class=\"amendement\">Amendement</li>\t\t<li class=\"beantwoording-raadsvragen\">Beantwoording raadsvragen</li>\t\t<li class=\"brief\">Brief</li>\t\t<li class=\"brieven-van-buiten\">Brieven van buiten</li>\t\t<li class=\"documentsoorten\">Documentsoorten</li>\t\t<li class=\"informatie\">Informatie</li>\t\t<li class=\"ingekomen-brieven\">ingekomen brieven</li>\t\t<li class=\"ingekomen-stuk\">Ingekomen stuk</li>\t\t<li class=\"ingekomen-stukken\">Ingekomen stukken</li>\t\t<li class=\"inhoud-map-losse-stukken\">Inhoud map losse stukken</li>\t\t<li class=\"kennisgeving\">Kennisgeving</li>\t\t<li class=\"lange-termijn-agenda\">Lange termijn agenda</li>\t\t<li class=\"mededelingen\">Mededelingen</li>\t\t<li class=\"memo\">Memo</li>\t\t<li class=\"notitie\">Motie</li>\t\t<li class=\"nieuwsbrief\">Nieuwsbrief</li>\t\t<li class=\"nieuwsbrieven\">Nieuwsbrieven</li>\t\t<li class=\"nota\">Nota</li>\t\t<li class=\"onbekend\">Onbekend</li>\t\t<li class=\"openbare-besluitenlijst-bw\">Openbare besluitenlijst B&amp;W</li>\t\t<li class=\"overig\">Overig</li>\t\t<li class=\"overige-bijlagen\">Overige bijlagen</li>\t\t<li class=\"overzicht-toezeggingen\">Overzicht toezeggingen</li>\t\t<li class=\"raadsbesluit\">Raadsbesluit</li>\t\t<li class=\"raadsinformatiebrief\">Raadsinformatiebrief</li>\t\t<li class=\"raadvoorstel\">Raadsvoorstel</li>\t\t<li class=\"speciale documenten\">Speciale Documenten</li>\t\t<li class=\"stukken-ter-inzage\">Stukken ter inzage</li>\t\t<li class=\"stukken-ter-kennisname\">Stukken ter kennisname</li>\t\t<li class=\"tvb\">TVB</li>\t\t<li class=\"uitnodiging\">Uitnodiging</li>\t\t<li class=\"uitnodigingen\">Uitnodigingen</li>\t\t<li class=\"vergaderschema\">Vergaderschema</li>\t\t<li class=\"verordening\">verordening</li>\t\t<li class=\"verslag\">Verslag</li>\t\t<li class=\"vragen-van-raadsleden\">Vragen van raadsleden</li>\t\t<li class=\"wandelgang\">Wandelgang</li>\t\t<li class=\"vertrouwlijk\">Vertrouwelijk document</li>\t\n</ul>\n\n</div>\r\n\t\t\t\t\t\t</div>\r\n\t\t\t\t\t\t\r\n\t\t\t\t\t\t<span class=\"clearer\">&nbsp;</span>\r\n\t\t\t\t\t</div>\r\n\t\t\t\t\t\r\n\t\t\t\t\t\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"documenten\">\n<h2>Vergaderstukken:</h2>\n<ul>\n\t<li class=\"onbekend\">\n\t<a href=\"Oproep-vergadering-gemeenteraad-9.pdf\">Oproep vergadering gemeenteraad.pdf</a>\n\t<span class=\"extention pdf\">(pdf, 486.13 kb)</span>\n\t<a href=\"Oproep-vergadering-gemeenteraad-9.pdf/notitie/\" title=\"Notitie\" class=\"notelink \" id=\"notelink17667_0_\">\n\t\t<span class=\"nonvisual\">notitie</span>\n\t</a>\n</li>\n</ul>\n</div>\n\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"agendapunten\">\r\n\t\t\t\t\t\t<h2>Agenda</h2>\n<ul id=\"vergadering\" class=\"hilite\">\n<li id=\"agendapunt17672_0\" class=\"actief agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\">1</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\"> Opening.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17672_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17672\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Opening-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_17672\" class=\"attachementRow attachement\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17672\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17672\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17672\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_17672\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17672\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17672\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_17672\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17672_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19254_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">2</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Verslag commissie onderzoek geloofsbrieven en be\u00ebdiging en installatie van de benoemde raadsleden, mevrouw A. Hogendoorn en de heren R. van Deutekom, A.A.H. Koenen en A.J. Pruiksma.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19254_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19254\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19254\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19254\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19254\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19254\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19254\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19254\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19254\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19254\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19254_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17671_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\">3</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\"> Spreekrecht burgers.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17671_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17671\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Spreekrecht-burgers\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_17671\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17671\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17671\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17671\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_17671\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17671\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17671\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_17671\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17671_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17670_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\">4</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\"> Vragenkwartier.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17670_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17670\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vragenkwartier-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_17670\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17670\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17670\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17670\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_17670\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17670\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17670\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_17670\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17670_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17669_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\">5</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\"> Bepalen stemvolgorde.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17669_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17669\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Bepalen-stemvolgorde\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_17669\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17669\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17669\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17669\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_17669\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17669\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17669\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_17669\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17669_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17668_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\">6</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\"> Vaststelling agenda.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17668_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button17668\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Vaststelling-agenda-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_17668\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17668\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17668\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17668\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_17668\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17668\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17668\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_17668\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17668_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19198_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">7</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het wijzigen van de Algemene plaatselijke verordening 2012 voor het gebruik van knalapparatuur in de landbouw.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Voorgesteld wordt het in de APV gebezigde ontheffingsstelsel voor het gebruik van knalapparatuur voor het verjagen van vogels en wild in de landbouw te vervangen door een stelsel met algemene regels.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19198_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19198\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19198\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19198\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19198\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19198\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19198\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19198\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19198\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19198\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19198_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19245_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">8</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel met betrekking tot de wijziging van de Gemeenschappelijke Regeling Regionale Uitvoeringsdienst Noord-Holland Noord (RUD NHN).</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">De RUD NHN voert sinds 1 januari 2014 voornamelijk milieu- en bodemtaken uit voor de provincie en de gemeenten op basis van de Gemeenschappelijke Regeling Noord-Holland Noord. Naar aanleiding van een wijziging van de Wet gemeenschappelijke regelingen en wegens het feit dat nog een aantal punten in de regeling nader dienden te worden ingevuld, wordt de raad voorgesteld in te stemmen met een aantal aanpassingen van de gemeenschappelijke regeling.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19245_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19245\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19245\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19245\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19245\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19245\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19245\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19245\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19245\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19245\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19245_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19199_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\">9</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\"> Voorstel tot het vaststellen van de Kwijtscheldingsverordening 2015.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Per 1 januari 2015 is de uitvoering van de gemeentelijke belastingen overgegaan naar de Gemeenschappelijke Regeling Cosensus. Omdat Cosensus ook een deel van de invordering uitvoert waaronder de kwijtschelding van de gemeenteljike belastingen dient ook de kwijtscheldingsverordening geactualiseerd en opnieuw vastgesteld te worden.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19199_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19199\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-vaststellen-van-de-Kwijtscheldingsverordening-2015\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19199\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19199\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19199\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19199\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19199\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19199\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19199\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19199\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19199_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19233_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">10</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van de Welstandsnota 2015 en de voorgestelde wijziging van de reclamenota \"Bedrijven (positief) in beeld\".</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Op 17 maart 2014 heeft de raad de kaderstelling voor de herziening van de Welstandsnota vastgesteld. Thans wordt de raad voorgesteld de op deze kaderstelling hierziene nota, de Welstandsnota 2015, vast te stellen. De nota is helder en compact geworden met het accent op behoud en stimuleren van de ruimtelijke kwaliteit.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19233_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19233\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19233\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19233\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19233\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19233\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19233\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19233\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19233\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19233\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19233_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19253_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">11</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van de Hoofdlijnen van aanpak voor een duurzaam en klimaatbestendig Den Helder.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">De gemeente speelt een belangrijke rol bij het communiceren, stimuleren en faciliteren van initiatieven op het gebied van duurzaamheid en klimaatbestendigheid. De komende vier jaar worden alleen die zaken opgepakt waarop binnen deze termijn vooruitgang te maken is.  Het betreft duurzaam handelen bij ruimtelijke ontwikkelingen, energiebesparingsmaatregelen in de bestaande woningvoorraad en bij bedrijven, bodemsaneringen, afvalrecycling, duurzame economie stimuleren, het goede voorbeeld geven als gemeente, stimulering plaatselijke energieproductie, bewonersinitiatieven en bewustwordingsacties.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19253_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19253\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19253\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19253\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19253\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19253\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19253\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19253\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19253\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19253\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19253_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19247_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">12</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van het bestemmingsplan Huisduinen en de Stelling 2015.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Het bestemmingsplan voorziet in een actuele regeling voor het plangebied, waarin alle beleidsstukken zijn meegenomen. Tot het plangebied behoren ook de Stelling met de forten Erfprins en Dirksz Admiraal en de daarbij behorende voormalige schootsvelden.  In het bestemmingsplan wordt ook geanticipeerd op de wensen van de Huisduiner Ontwikkelings Maatschappij voor de ontwikkeling van het gebied Huisduinerkwartier. Er zijn acht zienswijzen op het ontwerp-bestemmingsplan ontvangen. Het voorstel is om deze zienswijzen gedeeltelijk te volgen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19247_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19247\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19247\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19247\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19247\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19247\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19247\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19247\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19247\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19247\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19247_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19246_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">13</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel met betrekking tot de tussenrapportage 2015.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Als onderdeel van de planning- en controlcyclus biedt het college van burgemeester en wethouders de tussenrapportage 2015 aan. De rapportage is gebaseerd op de periode januari tot en met augustus en geeft de voortgang en afwijkingen aan ten opzichte van de programmabegroting 2015. De raad dient de bijbehorende wijziging van de programmabegroting vast te stellen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19246_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19246\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19246\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19246\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19246\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19246\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19246\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19246\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19246\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19246\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19246_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19248_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">14</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het vaststellen van het Integraal Huisvestingsplan 2015-2030 met het daarbij behorende investeringsoverszicht en aanbevelingen.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Het Integraal Huisvestingsplan 2015-2030 laat zien hoe de gemeente de komende periode wil omgaan met de onderwijshuisvesting op het grondgebied van de gemeente. In het Integraal Huisvestingsplan wordt met name ingezet op de koers die met schoolbesturen is ingezet in het kader van de daling van het aantal leerlingen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19248_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19248\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19248\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19248\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19248\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19248\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19248\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19248\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19248\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19248\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19248_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19250_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\">15</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\"> Voorstel tot het voorzien in de gemeentelijke vertegenwoordiging in de besturen van de zes gemeenschappelijke regelingen.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19250_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19250\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Voorstel-tot-het-voorzien-in-de-gemeentelijke-vertegenwoordiging-in-de-besturen-van-de-zes-gemeenschappelijke-regelingen\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19250\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19250\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19250\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19250\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19250\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19250\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19250\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19250\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19250_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19251_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">16</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot benoeming van leden van de Noordkopraad.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19251_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19251\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19251\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19251\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19251\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19251\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19251\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19251\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19251\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19251\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19251_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19249_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">17</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het benoemen van leden in de auditcommissie.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19249_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19249\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19249\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19249\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19249\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19249\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19249\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19249\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19249\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19249\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19249_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19252_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">18</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot benoeming van een lid van de vertrouwenscommissie.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19252_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19252\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19252\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19252\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19252\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19252\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19252\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19252\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19252\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19252\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19252_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19234_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">19</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel voor herstel en herbestemming van het Logementsgebouw (het Casino) in Huisduinen.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">Het Logementsgebouw  is een zeer bijzonder gebouw binnen het oorlogserfgoed van Den Helder, maar ook binnen het oorlogserfgoed van Nederland. Het logementsgebouw vormt samen met de loods en het poortgebouw een monumentaal complex dat op de lijst van Rijksmonumenten is geplaatst. Het voornemen is het gebouw te ontwikkelen als Atlantikwall Expeditie Centrum, een bezoekerscentrum met een interactieve presentatie. Uitvalsbasis voor een bezoek aan de Stelling van Den Helder en de Atlantikwall in Den Helder en het Waddengebied. Voorgesteld wordt (financi\u00eble) ondersteuning te bieden, door cofinanciering van Waddenfondssubsidie.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19234_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19234\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19234\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19234\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19234\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19234\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19234\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19234\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19234\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19234\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19234_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt19235_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\">20</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\"> Voorstel tot het beschikbaar stellen van een krediet van \u20ac 4,7 mln. voor de verwerving van de benodigde gronden en de realisatie van de Noorderhaaks, inclusief deelprojecten.</a></h3>\n\t\t<em></em>\n\t\t<div class=\"toelichting\">De raad heeft het college opgedragen een voorstel aan de raad voor te leggen voor de aanleg van de Noorderhaaks. Het college stelt thans de raad voor een krediet van \u20ac 4,7 mln beschikbaar te stellen. De investering wordt deels bekostigd door een provinciale subsidie en deels vanuit gemeentelijke reserveringen.</div>\n\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30//notitie/\" title=\"Notitie\" class=\"\" id=\"notelink19235_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"Agendapunt bevat bijlagen\" class=\"bijlage_true\"><span>Bijlage</span></a>\n\t\t<a id=\"button19235\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_19235\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_19235\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_19235\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_19235\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_19235\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_19235\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_19235\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_19235\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note19235_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n<li id=\"agendapunt17667_0\" class=\" agendaRow\">\n\t<div class=\"first\">\n\t\t<a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\">21</a>\n\t</div>\n\t<div class=\"title\">\n\t\t<h3><a class=\"\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\"> Sluiting.</a></h3>\n\t\t<em></em>\n\t\t\n\t</div>\n\t<div class=\"last\">\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-/notitie/\" title=\"Notitie\" class=\"\" id=\"notelink17667_2_\"><span class=\"nonvisual\">notitie</span></a>\n\t\t\n\t\t\n\t\t\n\t\t<a href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\" title=\"Er zijn geen bijlagen bij dit agendapunt\" class=\"bijlage_false\"><span>Bijlage</span></a>\n\t\t<a id=\"button17667\" href=\"/Vergaderingen/Gemeenteraad/2015/12-oktober/19:30/Sluiting-\" title=\"\" class=\"normaal\"><span>normaal</span></a>\n\t</div>\n\t<div class=\"clearer\"></div>\n\n\t<div id=\"attachements_17667\" class=\"attachementRow attachement hide\">\n\t\t<div class=\"filmcontent\">\n\t\t\t<div id=\"film_17667\" class=\"film\">\n\t\t\t\t \n\t\t\t\t \n\t\t\t\t\n\t\t\t</div>\n\t\t\t<div id=\"fragmenten_17667\" class=\"fragments\"></div>\n\t\t</div>\n\t\t\n\t\t<div class=\"speakerfragments active\">\n\t\t\t<div id=\"aanhetwoord_17667\" class=\"speaking\"></div>\n\t\t\t\n\t\t\t<div id=\"persons_17667\" class=\"persons\">\n\t\t\t\t<div id=\"sprekers_17667\" class=\"speakers\"></div>\n\t\t\t</div>\n\t\t</div>\n\t\t\n\t\t<div class=\"info\">\n\t\t\t<div id=\"meerinformatie_17667\" class=\"more\"></div>\n\t\t\t<div id=\"agendapunt_documenten_17667\" class=\"documents\"></div>\n\t\t</div>\n\t\t<div class=\"clearer\"></div>\n\t\t\n\t</div><!--eof attachementRow-->\n\n<div id=\"note17667_2\" class=\"\">\n\t\n</div>\n\t\n</li><!-- eof agendaRow-->\n\n</ul><!--eof vergadering-->\n<input type=\"hidden\" name=\"fragment\" id=\"fragment\" value=\"0\" />\n\r\n\t\t\t\t\t</div>\r\n\t\t\t\t\t\r\n\t\t\t\t\t\r\n\t\t\t\t\t\r\n\t\t\t\t\t\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"status_uitzending\"></div>\r\n\t\t\t\t\t<div id=\"presentation\" class=\"leeg\">\r\n\t\t\t\t\t\t<img id=\"vergroten\" src=\"/md/zoomDoc.gif\" alt=\"afbeelding vergroten\" onclick=\"togglesize('#presentation_sheet')\" />\r\n                        <img id=\"presentation_sheet\" src=\"/md/empty.jpg\" title=\"klik om te vergroten/verkleinen\" alt=\"laden...\" class=\"normal_sheet\" onclick=\"togglesize(this)\" />\r\n\t\t\t\t\t</div>\r\n\t\t\t\t</div>\r\n\t\t\t\t\r\n\t\t\t\t<div class=\"clearer\"> </div>\r\n\t\t\t</div>\r\n\t\t\t\r\n\t\t\t<!-- Header -->\r\n\t\t\t<div id=\"header\">\r\n\t\t\t\t<div class=\"container\">\r\n\t\t\t\t\t<div id=\"logo\"><a href=\"/\" title=\"Terug naar home\">Gemeente Den Helder</a></div>\r\n\t\t\t\t\t\r\n\t\t\t\t\t<div id=\"options\"><div class=\"options_wrapper\">\r\n\t<h3 class=\"nonvisual\">Pagina opties</h3>\r\n\t<ul>\r\n\t\t<li><a class=\"rss\" href=\"/rss_vergaderingen\">RSS</a></li>\r\n\t\t<li><a class=\"print\" href=\"/vergaderingen/Gemeenteraad/2015/12-oktober/19:30/print\">Afdrukken</a></li>\r\n\t\t<li><a class=\"readspeak\" href=\"/\">Lees voor</a></li>\r\n\t\t<li>\r\n\t\t\t<a class=\"text_sml\" href=\"/zoom/level1\" title=\"Kleinere letters\"><span class=\"nonvisual\">Kleinere letters</span></a>\r\n\t\t\t<a class=\"text_mid\" href=\"/zoom/level2\" title=\"Normale letters\"><span class=\"nonvisual\">Normale letters</span></a>\r\n\t\t\t<a class=\"text_lrg\" href=\"/zoom/level3\" title=\"Grote letters\"><span class=\"nonvisual\">Grote letters</span></a>\r\n\t\t</li>\r\n\t</ul>\r\n</div></div>\r\n\t\t\r\n\t\t\t\t\t<div id=\"search\"><h3 class=\"nonvisual\" id=\"zoeken\">Zoeken</h3>\r\n<form action=\"/zoek/\" method=\"post\">\r\n\t<fieldset>\r\n\t\t<legend>Snelzoeken</legend>\r\n\t\t<label class=\"nonvisual\" for=\"q\">Zoekterm</label>\r\n\t\t<input type=\"text\" id=\"q\" name=\"zoek_opdracht\" value=\"Zoeken...\" />\r\n\t\t<input type=\"submit\" id=\"search_button\" name=\"vind\" value=\"Zoek\" />\r\n\t\t<input type=\"hidden\" value=\"alle_woorden\" class=\"hidden\" name=\"search\" />\r\n\t</fieldset>\r\n</form>\r\n</div>\r\n\t\t\t\t</div>\r\n\t\t\t\t\r\n\t\t\t\t<div id=\"menuHolder\">\r\n\t\t\t\t\t<div class=\"container\">\r\n\t\t\t\t\t\t<h2 class=\"nonvisual\">Site Navigatie</h2>\r\n\t\t\t\t\t\t<a class=\"skip\" href=\"#header\">Site Navigatie overslaan</a>\r\n\t\t\t\t\t\t<h3 class=\"nonvisual\">Hoofdmenu</h3>\n<ul id=\"menu\">\n<li><a href=\"/\"><span>raad home</span></a></li>\n<li><a href=\"/Actueel\"><span>actueel</span></a></li>\n<li><a href=\"/Organisatie\"><span>organisatie</span></a></li>\n<li><a class=\"active\" href=\"/Vergaderingen\"><span>vergaderingen</span></a></li>\n<li><a href=\"/Documenten\"><span>documenten</span></a></li>\n<li><a href=\"/Abonnement\"><span>abonnement</span></a></li>\n<li><a href=\"/Kalender\"><span>kalender</span></a></li>\n</ul>\t\r\n\t\t\t\t\t</div>\r\n\t\t\t\t</div>\r\n\t\t\t\t\r\n\t\t\t\t<div id=\"user\">\r\n\t\t\t\t\t<div class=\"container\"> <h3 class=\"nonvisual\">Inloggen</h3>\n<p class=\"nonvisual\">U kunt inloggen als raadslid: </p><a id=\"inloggen_link\" href=\"/inloggen?url=/vergaderingen/Gemeenteraad/2015/12-oktober/19:30\">Inloggen</a>\n </div>\r\n\t\t\t\t</div>\r\n\t\t\t</div>\r\n\t\t\t\r\n\t\t</div>\r\n        <!-- Footer -->\r\n        <div id=\"footer\">\r\n            <div class=\"container\">\r\n\t<div class=\"block\">\r\n\t\t<h3>Contact</h3>\r\n\t\t<address>\r\n\t\t\t<p>\r\n\t\t\t\t<span>Raadsgriffie gemeente Den Helder</span>\r\n\t\t\t\t<span>Drs. F. Bijlweg 20</span>\r\n\t\t\t\t<span>Postbus 36</span>\r\n\t\t\t\t<span>1780 AA, Den Helder</span>\r\n\t\t\t</p>\r\n\t\t\t\r\n\t\t\t<p>\r\n\t\t\t\t<span>T: 14 0223</span>\r\n\t\t\t\t<span>F: (0223) 67 12 01</span>\r\n\t\t\t\t<span><a href=\"mailto:griffie@denhelder.nl\" title=\"Mail naar gemeente Den Helder\">griffie@denhelder.nl</a></span>\r\n\t\t\t</p>\t\t\r\n\t\t</address>\r\n\t</div>\r\n\t\r\n\t<div class=\"block\">\r\n\t\t<h3>Over deze site</h3>\r\n\t\t<ul>\r\n\t\t\t<li><a class=\"home\" href=\"/\"><span>Home</span></a></li>\r\n\t\t\t<li><a class=\"contact\" href=\"/Over-deze-site/Contact/\">Contact</a></li>\r\n\t\t\t<li><a class=\"disclaimer\" href=\"/Over-deze-site/Disclaimer/\">Disclaimer</a></li>\r\n\t\t\t<li><a class=\"colofon\" href=\"/Over-deze-site/Colofon/\">Colofon</a></li>\t\r\n\t\t\t<li><a class=\"toegankelijkheid\" href=\"/Over-deze-site/Toegankelijkheid/\">Toegankelijkheid</a></li>\t\r\n\t\t\t<li><a class=\"sitemap\" href=\"/Over-deze-site/Sitemap/\">Sitemap</a></li>\t\r\n\t\t\t<li><a class=\"zoeken\" href=\"/zoeken\">Zoeken</a></li>\t\r\n\t\t</ul>\r\n\t</div>\r\n\t\r\n\t<div class=\"block\">\r\n\t\t<h3>Volg ons op</h3>\r\n\t\t<ul>\r\n\t\t\t<li><a class=\"rss\" href=\"/rss\"><span>RSS</span></a></li>\r\n\t\t</ul>\r\n\t\t<br>\r\n\t\t<h3>Gemeentelijke website</h3>\r\n\t\t<ul>\r\n\t\t\t<li><a href=\"http://www.denhelder.nl\">Gemeente Den Helder</a></li>\r\n\t\t</ul>\r\n\t</div>\r\n\t\r\n\t<a href=\"#top\" id=\"toTop\">Terug naar boven</a>\r\n\t\r\n</div>\r\n<div id=\"copy\">\r\n\t<p class=\"container\">\r\n\t\t<span>&copy;2015 Gemeente Den Helder</span>\r\n\t\t<a href=\"https://www.gemeenteoplossingen.nl/producten/papierloos_werken/go_raadsinformatie/\" title=\"Meer weten over deze dienst?\" class=\"right\">GemeenteOplossingen | Raadsinformatie</a>\r\n\t</p>\r\n</div>\r\n        </div>\r\n        \t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery.js\"></script>\r\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery-ui.js\"></script>\r\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery.cookie.js\"></script>\r\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/jquery.ui.datepicker-nl.js\"></script>\r\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/activeplx.js\"></script>\r\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/gui.js\"></script>\r\n\t\t<script type=\"text/javascript\" src=\"/site/default2014/script/vergadering.js\"></script>\r\n\r\n        \r\n        <script type=\"text/javascript\" src=\"/site/default2014/script/prototype.js\"></script>\r\n\r\n        <script type=\"text/javascript\" src=\"/jwplayer/jwplayer.js\"></script>\r\n        <script type=\"text/javascript\">jwplayer.key=\"UCSfwBt+qmRTLBgiLL70hAT6TP3dtA29Yg7HEw==\";</script>\r\n\t</body>\r\n</html>\r\n<!-- t:-0.598804 -->", "type": "meeting"}

.. todo::

    - The stats functionality is currently not working

.. http:get:: /(source_id)/(doc_type)/(object_id)/stats

   Retrieves statistics about the usage of the object within the Open Raadsinformatie API.

   :statuscode 200: OK, no errors.
   :statuscode 404: The requested source and/or object does not exist.

   **Example request**

   .. sourcecode:: http

      $ curl -i 'http://api.openraadsinformatie.nl/v0/den_helder/events/4a39c497c7818af9ad6d7d8335a6e951d6a83bd6/stats'

   **Example response**

   .. sourcecode:: http

      HTTP/1.1 200 OK
      content-type: application/json
      content-length: 115
      date: Wed, 27 May 2015 13:00:24 GMT

      {
        "n_appeared_in_search_results": 0,
        "n_appeared_in_similar_results": 0,
        "n_get": 0,
        "n_get_source": 0
      }

.. _rest_similar:

Similar items
-------------

.. http:post:: /similar/(object_id)

  Retrieve objects similar to the object with id ``object_id`` across all indexed datasets (i.e. it could return items with similar descriptions). From the contents of the object, the most descriptive terms ("descriptive" here means the terms with the highest tf-idf value in the document) are used to search across the data sources.

  As a search is executed, the response format is exactly the same as the response returned by the :ref:`search endpoint <rest_search>`. The request format is almost the same, with the exception that a query can't be specified (as the document with id ``object_id`` is considered the query). That means that faceting, filtering and sorting on the resulting set are fully supported.

  :jsonparameter filters: an object with field and values to filter on (optional).
  :jsonparameter facets: an object with fields for which to return facets (optional).
  :jsonparameter sort: the field the search results are sorted on. By default, results are sorted by relevancy to the query.
  :jsonparameter size: the maximum number of documents to return (optional, defaults to 10).
  :jsonparameter from: the offset from the first result (optional, defaults to 0).
  :statuscode 200: OK, no errors.
  :statuscode 400: Bad Request. An accompanying error message will explain why the request was invalid.

  **Example request**

  .. sourcecode:: http

    $ curl -i -XPOST 'http://api.openraadsinformatie.nl/v0/similar/<object_id>' -d '{
       "facets": {
          "classification": {}
       },
       "size": 10
    }'


.. http:post:: /(source_id)/similar/(object_id)

  Retrieve objects similar to the object with id ``object_id`` from the dataset specified by ``source_id``. You can find similar objects in the same data source, or objects in a different data sources that are similar to the provided object.

  :jsonparameter filters: an object with field and values to filter on (optional).
  :jsonparameter facets: an object with fields for which to return facets (optional).
  :jsonparameter sort: the field the search results are sorted on. By default, results are sorted by relevancy to the query.
  :jsonparameter size: the maximum number of documents to return (optional, defaults to 10).
  :jsonparameter from: the offset from the first result (optional, defaults to 0).
  :statuscode 200: OK, no errors.
  :statuscode 400: Bad Request. An accompanying error message will explain why the request was invalid.

.. http:post:: /(source_id)/(doc_type)/similar/(object_id)

  Retrieve objects similar to the object with id ``object_id`` from the dataset specified by ``source_id``, limited by a cerrtain document type (persons, organizations or events). You can find similar objects in the same data source, or objects in a different data sources that are similar to the provided object with the same document type.

  :jsonparameter filters: an object with field and values to filter on (optional).
  :jsonparameter facets: an object with fields for which to return facets (optional).
  :jsonparameter sort: the field the search results are sorted on. By default, results are sorted by relevancy to the query.
  :jsonparameter size: the maximum number of documents to return (optional, defaults to 10).
  :jsonparameter from: the offset from the first result (optional, defaults to 0).
  :statuscode 200: OK, no errors.
  :statuscode 400: Bad Request. An accompanying error message will explain why the request was invalid.

.. _rest_resolver:

Resolver
--------
The Open Raadsinformatie API provides all (media) urls as Resolver URLs. This will route all requests for content through the API, which will process and validate the URL, and provide a redirect to the original content source. This will allow for caching or rate limiting on API level in the future, to prevent excessive amounts of requests to the sources.

.. http:get:: /resolve/(url_hash)

  Resolves the provided URL, returns the content if it is valid. When the resource is not available locally in the cache, the request is redirected with a 302. If it is not, a 404 is returned. Depending on the Accept header in the request, it returns a JSON-encoded response detailing what went wrong, or a HTML-page, allowing for transparent use in websites.

  :statuscode 200: Resource is cached and is returned tot the client as MIME-content.
  :statuscode 302: Redirecting to location of the resource if it is not locally cached.
  :statuscode 404: The resource was not found.


  **Example json request**

  .. sourcecode:: http

      $ curl -i -Haccept:application/json -XGET http://www.openraadsinformatie.nl/v0/resolve/<url_hash>

  **Example browser-like request**

  .. sourcecode:: http

      $ curl -i -Haccept:text/html -XGET http://www.openraadsinformatie.nl/v0/resolve/<url_hash>

  **Example success response**

  .. sourcecode:: http

      HTTP/1.0 302 Found
      Location: http://example.com/example.jpg

  .. sourcecode:: http

      HTTP/1.0 302 FOUND
      Location: http://<STATIC_SUB_DOMAIN>.openstate.eu/media/<img_name>.jpg"

  **Example failed json response**

  .. sourcecode:: http

      HTTP/1.0 404 NOT FOUND
      Content-Type: application/json
      Content-Length: 98
      Date: Sat, 24 May 2014 14:33:00 GMT

      {
        "error": "URL is not available; the source may no longer be available",
        "status": "error"
      }

  **Example failed HTML response**

  .. sourcecode:: http

      HTTP/1.0 404 NOT FOUND
      Content-Type: text/html; charset=utf-8
      Content-Length: 123
      Date: Sat, 24 May 2014 14:32:37 GMT

      <html>
        <body>
          There is no original url available. You may have an outdated URL, or the resolve id is incorrect.
        </body>
      </html>
