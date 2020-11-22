# Async/await begrijpen

Als je niet bekend bent met ECMAScript 2017, zal je waarschijnlijk niet bekend zijn met async/await. Het is een handige manier om Promises te verwerken. Daarnaast ziet het er beter uit, is het beter leesbaar en is het ook iets sneller.

## Hoe werken Promises?

Voordat we kunnen beginnen met async/await, moet je weten wat Promises zijn en hoe ze werken. Als je dit al weet, kan je dit onderdeel overslaan.

Promises zijn een manier om asynchrone taken uit te voeren in Javascript; ze zijn een nieuwe alternatief tegenover callbacks. Een Promise lijkt een beetje op een laadbalk; ze vertegenwoordigen een proces dat nog niet klaar is. Een goed voorbeeld daarvan is een verzoek aan een server. (b.v. discord.js stuurt verzoeken naar Discord's API).

Een Promise kan zich in 3 staten bevinden; *pending*, *resolved*, en *rejected*

De staat **pending** betekent dat de Promise nog bezig is, en dus niet gefaald of gelukt is.
De staat **resolved** betekent dat de Promise klaar is en zonder foutmeldingen is afgerond.
De staat **rejected** betekent dat de Promise een foutmelding heeft veroorzaakt en niet goed kon worden afgerond.

Een belangrijk onderdeel om te onthouden is dat een Promise altijd maar in 1 staat kan zijn; hij kan nooit pending en resolved zijn, rejected en resolved, of pending en rejected. Je vraagt je misschien af "Hoe ziet dat eruit in code?". Hier is een klein voorbeeld:

::: tip
Deze code gebruikt ES6. Als je niet weet wat dat is, kan je [hier](/additional-info/es6-syntax.md) meer lezen.
:::

```js
function deleteMessages(aantal) {
	return new Promise(resolve => {
		if (aantal > 10) throw new Error('Je kan niet meer dan 10 berichten verwijderen.');
		setTimeout(() => resolve('10 berichten verwijderd.'), 2000);
	});
}

deleteMessages(5).then(value => {
	// `deleteMessages` is compleet en heeft geen foutmeldingen veroorzaakt
	// 'value' zal de string "10 berichten verwijderd" zijn.
}).catch(error => {
	// `deleteMessages` heeft een foutmelding veroorzaakt
	// 'error' zal een Error object zijn
});
```

In dit scenario, retourneert de `deleteMessages` functie een Promise. De methode `.then()` zal afgaan als de Promise de staat 'resolved' bereikt, en de `.catch()` methode als bij de 'rejected' staat. Maar, met onze functie resolven we de Promise na 2 seconden met de string "10 berichten verwijderd.", dus de `.catch()` methode zal nooit afgaan. Je kan de `.catch()` functie ook als tweede parameter van `.then()` gebruiken.

## Hoe implementeer je async/await?

### Theorie

Het volgende is belangrijk om te weten als je gaat werken met async/await. Je kan het `await` trefwoord alleen gebruiken in een functie die gedeclareerd is als `async` (je zet het `async` trefwoord voor het `function` trefwoord, of voor de parameters als je een callback functie gebruikt). 

Een simpel voorbeeld zou zijn:

```js
async function gedeclareerdAlsAsync() {
	// code
}
```

or

```js 
const gedeclareerdAlsAsync = async () => {
	// code
};
```

Je kan die laatste ook gebruiken bij een pijlfunctie;

```js
client.on('event', async (first, last) => {
	// code
});
```

Iets belangrijks om te weten is dat een functie gedeclareerd als `async` altijd een Promise zal retourneren. Hiernaast zal, als je iets retourneert, de Promise hiermee resolven. Als je een foutmelding gooit, zal deze rejecten met deze foutmelding.

### Gebruik met discord.js code

Nu dat je weet hoe Promises werken, laten we gaan kijken naar een voorbeeld waarin we meerdere Promises zullen verwerken. Stel dat je wilt reageren met letters (regionale indicatoren) in een bepaalde volgorde. In dit voorbeeld zal je het basissjabloon voor een discord.js bot met een paar ES6 wijzigingen gebruiken.

```js
const Discord = require('discord.js');
const client = new Discord.Client();

const prefix = '?';

client.once('ready', () => {
	console.log('Klaar voor gebruik');
});

client.on('message', message => {
	if (message.content === `${prefix}reageer`) {
		// Code hier
	}
});

client.login('token');
```

Dus nu moeten we de code invoeren. Als je niet weet hoe Node.js asynchrone executie werkt, zou je waarschijnlijk zoiets proberen:

```js
client.on('message', message => {
	if (message.content === `${prefix}reageer`) {
		message.react('🇦');
		message.react('🇧');
		message.react('🇨');
	}
});
```
Maar omdat al deze `react()` methodes op hetzelfde moment starten, wordt het een race om welk verzoek het snelste afgehandeld wordt, dus is er geen garantie dat de reacties worden toegevoegd in deze volgorde. Om ervoor te zorgen dat het in de goede volgorde wordt uitgevoerd, moeten we de `.then()` callback gebruiken. Als resultaat zou de code er ongeveer zo uitzien:

```js
client.on('message', message => {
	if (message.content === `${prefix}reageer`) {
		message.react('🇦')
			.then(() => message.react('🇧'))
			.then(() => message.react('🇨'))
			.catch(error => {
				// Als er iets fout gaat, wordt dat hier verwerkt
			});
	}
});
```

In deze code [chain resolven](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then#Chaining) Promises met elkaar, en als een van de Promises gereject wordt, wordt de functie die we in `.catch()` gezet hebben gebruikt. Laten we kijken hoe dit eruit ziet met async/await.

```js
client.on('message', async message => {
	if (message.content === `${prefix}reageer`) {
		await message.react('🇦');
		await message.react('🇧');
		await message.react('🇨');
	}
});
```

Dat is ongeveer dezelfde code, maar hoe pakken we Promise rejections aan zonder `.catch()`? Dat is ook een handige eigenschap van async/await. De foutmelding zal gegooid worden als je het await, dus je kan gewoon de Promises in een `try`/`catch` blok zetten.

```js
client.on('message', async message => {
	if (message.content === `${prefix}reageer`) {
		try {
			await message.react('🇦');
			await message.react('🇧');
			await message.react('🇨');
		} catch (error) {
			// Als er iets fout gaat, wordt dat hier verwerkt
		}
	}
});
```

Dit ziet er mooi uit, en is ook makkelijk te lezen.

Je vraagt je misschien af, "hoe krijg ik de waarde waarmee de Promise ge-resolved is?"

Laten we kijken naar een voorbeeld waar je een verzonden bericht wilt verwijderen.

<branch version="11.x">

```js
client.on('message', message => {
	if (message.content === `${prefix}verwijder`) {
		message.channel.send('Dit bericht zal na 10 seconden verwijderd worden')
			.then(sentMessage => sentMessage.delete(10000))
			.catch(error => {
				// Foutmelding verwerken
			});
	}
});
```

</branch>
<branch version="12.x">

```js
client.on('message', message => {
	if (message.content === `${prefix}verwijder`) {
		message.channel.send('Dit bericht zal na 10 seconden verwijderd worden')
			.then(sentMessage => sentMessage.delete({ timeout: 10000 }))
			.catch(error => {
				// Foutmelding verwerken
			});
	}
});
```

</branch>
De geretourneerde waarde van `.send()` is een Promise dat resolvet naar een Message object. Maar hoe zou dit eruit zien in async/await?

<branch version="11.x">

```js
client.on('message', async message => {
	if (message.content === `${prefix}verwijder`) {
		try {
			const sentMessage = await message.channel.send('Dit bericht zal na 10 seconden verwijderd worden.');
			await sentMessage.delete(10000);
		} catch (error) {
			// Foutmelding verwerken
		}
	}
});
```

</branch>
<branch version="12.x">

```js
client.on('message', async message => {
	if (message.content === `${prefix}verwijder`) {
		try {
			const sentMessage = await message.channel.send('Dit bericht zal na 10 seconden verwijderd worden.');
			await sentMessage.delete({ timeout: 10000 });
		} catch (error) {
			// Foutmelding verwerken
		}
	}
});
```

</branch>

Met async/await kan je gewoon de functie in een variabele zetten, dat de geretourneerde waarde zal vertegenwoordigen. Nu weet je hoe je async/await moet gebruiken.
