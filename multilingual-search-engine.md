
Websites with a lot of user generated content lately started to adopt and support multiple languages! I think this is great, because basically, Why not? I mean this should have been the case from the beginning of the Internet, right?  But anyway, having this, some of them rely on "listings" textual content like: "title", "tags", "description" to index the whole content, and then offer an internal search engine for the follow visitors.

Among them, we have these websites that rely on the media content (images and videos) which are the raw value delivered to the user (I guess you already guessed what I'm talking about :o ). So a visitor looking for "A slim vs. a fat wrestling" :o , he/she will be interested in all media whatever their title, tags and description. This case particularly is on the rise, and if you have such a website and didn't start supporting multilingual search, then this is a starting point.

First we will be using **NodeJS** as a web server and **MongoDB** as a database. This is meant very basic just to illustrate how you would benefit from our **Word to Word API**.


```js
    // define your base query
    const baseQuery = { d: false, a: true }
    /**
     * Approximate multilingual search based on indexed text fields: title, desc, tags
     * @param {*} phrase sentence to search
     * @param {*} exact whether search the exact sentence or separate terms
     * @param {*} otherFields Of course you can add any other categorical fields to search
     * @param {*} lang which language the phrase is in
     * @return {Promise}
     */
    this.gwoogl = async function (phrase, exact, otherFields, lang) {
        // Limit search to newer content (ignoring very old content)
        const daysBefore = 100
        collection = mongoDB.collection('listing')
        const since = getObjectId(daysBefore)
        // quoted string value in Mongo to look for the exact match ! 
        phrase = exact ? `"${phrase.trim()}"` : phrase.trim()
        const query = JSON.parse(JSON.stringify(baseQuery)) // clone
        // Collation is very important, it helps Mongo with searching natural languages
        // something like "a car" can spot either "the car", "cars" etc
        // Note that Collation at the moment is an index on the whole collection, ie
        // You cannot apply different collation(s) on documents
        let collation = lang === 'und' ? baseCollation : { locale: lang }

        // THE FOLLOWING IS AN EXAMPLE 
        // HERE I'M FIRST SEARCHING FOR THE WHOLE PHRASE AS IS
        // THEN I CALL TRANSLATION API ON THE FIRST TERM IN THE PHRASE. YOU CAN ITERATE ON EACH WORD IN THE SAME MANNER !!
        // STACK AND FORMAT RESULTS AS YOU WISH !

        query.$text = { $search: phrase }
        query._id = { $gt: since }
        if (lang !== 'und') query.lang = lang
        // Dynamically adding other fields
        for (const [key, value] of Object.entries(otherFields)) {
            if (value !== 'und') query[key] = value
        }
        const docs = await collection
            .find(query, { score: { $meta: 'textScore' } })
            .collation(collation)
            .project(baseProjection)
            .sort({ score: { $meta: 'textScore' } })
            .limit(21)
            .toArray()
        const count = await collection.countDocuments(query)
        const result = { documents: docs, count: count, crossLangDocs: [] }

        if (count < 6 && phrase.indexOf(' ') < 0) {
            // Call translate API on each keyword, grab each query result (keywords), concatenate them as a single phrase and query again
            let translations = translate({ limit: 10, source: lang, score: '0.5', word: phrase, target: 'fr', target: 'es', target: 'fi' })
            for (const [lang, keywords] of Object.entries(translations)) {
                collation = { locale: lang }
                // Concatenating translations as a single phrase (you can do however you feel better) 
                phrase = keywords.join(' ')
                query.$text = { $search: phrase }
                const crossLangDocs = await collection
                    .find(query, { score: { $meta: 'textScore' } })
                    .collation(collation)
                    .project(baseProjection)
                    .sort({ score: { $meta: 'textScore' } })
                    .limit(3)
                    .toArray()
                // console.log(crossLangDocs)
                crossLangDocs.forEach((doc) => {
                    doc['crosslang'] = lang
                })
                result.crossLangDocs = result.crossLangDocs.concat(crossLangDocs)
            }
        }
        return result
    }
```


```js
// Helpers

/**
 * This function returns an ObjectId embedded with a given dateTime
 * Accepts number of days since document was created
 * Author: https://stackoverflow.com/a/8753670/1951298
 * @param {Number} days
 * @return {object}
 */
function getObjectId(days) {
    const yesterday = new Date()
    days = days || (process.env.NODE_ENV === 'localhost' ? 1000 : 14)
    yesterday.setDate(yesterday.getDate() - days)
    const hexSeconds = Math.floor(yesterday / 1000).toString(16)
    return new ObjectId(hexSeconds + '0000000000000000')
}

```

## Consider
```js
    // Indexing title and description fields (ie fields to search for)
    // collection#text must be indexed as the following:
    const listingCollection = db.collection('listing')
    await listingCollection.dropIndexes()
    await listingCollection.createIndex({ title: 'text', cdesc: 'text' }, { weights: { title: 3, cdesc: 1 } })

```

```js
// APIWrapper.js
// Requst Word to Word translator API

/*
* API wrapper
*
*/

import axios from "axios";

const translateParameters = (params) => {
    return {
        method: 'GET',
        url: 'https://word-to-word-translator1.p.rapidapi.com/api/translate',
        params,
        headers: { 'X-RapidAPI-Key': 'YOUR RAPID API GOES HERE' }
    }
};

const detectParameters = (params) => {
    return {
        method: 'GET',
        url: 'https://word-to-word-translator1.p.rapidapi.com/api/detect_language',
        params,
        headers: { 'X-RapidAPI-Key': 'YOUR RAPID API GOES HERE' }
    }
};


const translate = async (params) => {
    return await axios.request(translateParameters(params))
}

const detect = async (params) => {
    return await axios.request(detectParameters(params))
}

export { translate, detect }
```

## Final notes

You can see it is not straight forward to build a simple search engine, because again it involves natural languages. But at the same time, a performant solution built "in-house" is very possible!! 
A scenario to use our very tiny **Gwoogl** search engine is to:

1- Get user input (which can be real-time (frankly speaking, I think it is possible, but I personally did not try that to check performance))
2- Maybe do some data cleansing and adaptations (stemming, lemmatization etc). Please note that thankfully, our language detection and translation is very strong in this regard, it uses contextual search so it is fine for instance to leave the plural form !
3- Detect input language (`Word2Word#detect`)
4- Translate the input words (`Word2Word#translate`)
5- Iterate over keywords and stack results !

Hooray, now you have a multilingual search engine!
