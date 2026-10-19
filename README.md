# 12 Tale til tekst og oplæsning

Denne øvelse bygger videre på jeres eksisterende chatbot fra [10 AI Code](https://github.com/CBS-INNT-26/10_AI_code) og [11 AI Træning](https://github.com/CBS-INNT-26/11_AI_Traening_vejledning). Vi går ud fra, at appen allerede kan sende beskeder til en AI-tutor, og at `runGuardrails` kontrollerer hver brugerbesked.

I skal give tutoren **ører og en stemme**:

- Brugeren trykker på en mikrofon, taler og trykker igen. Appen skriver talen om til tekst.
- Den transskriberede tekst sendes som en almindelig besked i chatten. Den skal gennem de samme guardrails som skrevet tekst.
- Tutorens svar vises som før. Når oplæsning er slået til, læser appen det også højt.

**Start i jeres eget projekt fra øvelse 11.** Den komplette kode ligger i [11 AI Træning](https://github.com/CBS-INNT-26/11_AI_Traening_vejledning). Brug den som facit, hvis I går i stå. Herunder bygger vi funktionerne trin for trin, ligesom i øvelse 11.

---

## Sådan rejser beskeden gennem appen

```text
Brugeren taler
    ↓
expo-audio optager en m4a-fil på telefonen
    ↓
OpenAI /v1/audio/transcriptions (Whisper) laver lyd til tekst
    ↓
onSend → handleIncomingMessage → runGuardrails → getBardResp
    ↓
respondWithBotMessage viser tutorens svar
    ↓
OpenAI /v1/audio/speech laver tekst til mp3
    ↓
expo-audio spiller svaret i telefonens højttaler
```

Læg især mærke til midten: **tale skal ind gennem `onSend`**. I skal ikke lave en ny vej uden om `runGuardrails`. Hvis en talt besked er for lang, stødende eller uden for emnet, skal den afvises præcis som en skrevet besked.

| Del | Hvad gør den? | Hvor ligger den? |
| --- | --- | --- |
| `useAudioRecorder` | Optager lyd på telefonen | `screens/ChatScreen.js` |
| `transcribeAudio` | Sender lyd til Whisper og får tekst | `services/Voice.js` |
| `onSend` | Modtager en almindelig brugerbesked | Eksisterende `screens/ChatScreen.js` |
| `runGuardrails` | Kontrollerer teksten | Eksisterende `services/Guardrails.js` |
| `speakText` | Henter og afspiller mp3 | `services/Voice.js` |

De to OpenAI-kald bruger modellerne `whisper-1` til transskription og `gpt-4o-mini-tts` til oplæsning. Begge bruger den `OPENAI_API_KEY`, I allerede har i `@env`.

---

## 1. Installér lydpakkerne

Kør i roden af jeres Expo-projekt:

```bash
npx expo install expo-audio expo-file-system
```

`expo-audio` optager og afspiller. `expo-file-system` uploader optagelsen og gemmer mp3-svaret. Brug `npx expo install`, så versionerne passer til jeres Expo SDK. Ikonerne kommer fra `@expo/vector-icons`, der allerede bruges til send-knappen i øvelse 11.

I skal stadig have jeres egen `.env` i projektets rod:

```dotenv
OPENAI_API_KEY=din-egen-api-nøgle
```

`.env` skal være i `.gitignore`. Genstart Expo med `npx expo start -c`, hvis I netop har ændret nøglen.

### Mikrofontekst i app.json

Tilføj følgende element i den eksisterende `plugins`-liste i `app.json`:

```json
[
  "expo-audio",
  {
    "microphonePermission": "Appen bruger mikrofonen, så du kan tale med din tutor."
  }
]
```

Se den [færdige app.json](../12_Tale_til_tekst_code/app.json). Behold de plugins, I allerede har. Teksten bruges især, når I bygger appen som en rigtig iOS-app.

---

## 2. Lav services/Voice.js

I øvelse 11 lagde I API-logik i `services/Guardrails.js` og `services/Request.js`. Gør det samme med lyd, så skærmfilen kun styrer brugerens handlinger. Opret `services/Voice.js`.

### Importer det nødvendige

```javascript
import * as FileSystem from "expo-file-system/legacy";
import { File, Paths } from "expo-file-system";
import { fetch } from "expo/fetch";
import { createAudioPlayer, setAudioModeAsync } from "expo-audio";
import { OPENAI_API_KEY } from "@env";

const TRANSCRIBE_URL = "https://api.openai.com/v1/audio/transcriptions";
const SPEECH_URL = "https://api.openai.com/v1/audio/speech";

let currentPlayer = null;
```

Hvorfor to imports fra filsystemet? `uploadAsync` findes i `expo-file-system/legacy`. `File` og `Paths.cache` er den nyere måde at skrive mp3-filen til telefonens cache på. `currentPlayer` husker den afspiller, som skal kunne stoppes.

### 2.1 Tale til tekst

`transcribeAudio(uri)` får stien til telefonens optagelse. Uploaden skal være `MULTIPART`: lydfilen og formularfelterne sendes i samme anmodning. OpenAI forventer, at filfeltet hedder præcis `file`.

```javascript
export async function transcribeAudio(uri) {
  if (!uri) return null;

  try {
    const result = await FileSystem.uploadAsync(TRANSCRIBE_URL, uri, {
      httpMethod: "POST",
      uploadType: FileSystem.FileSystemUploadType.MULTIPART,
      fieldName: "file",
      mimeType: "audio/m4a",
      headers: { Authorization: `Bearer ${OPENAI_API_KEY}` },
      parameters: {
        model: "whisper-1",
        language: "???",
      },
    });

    const data = JSON.parse(result.body);
    if (result.status !== 200) {
      console.error("Fejl fra OpenAI (transskription):", data);
      return null;
    }
    return data.text?.trim() || null;
  } catch (error) {
    console.error("Fejl ved transskription:", error);
    return null;
  }
}
```

**Udfyld `???`:** Hvilken ISO-sprogkode på to bogstaver bruger man til dansk? Hvorfor kan det hjælpe modellen at kende sproget på forhånd?

Funktionen returnerer `null`, hvis der ikke kom nogen tekst, eller hvis kaldet fejlede. Dermed kan skærmen give brugeren en forståelig besked og undgå at sende en tom chatbesked.

### 2.2 Stop en oplæsning

Før I kan afspille, skal I kunne stoppe den forrige afspiller:

```javascript
export function stopSpeaking() {
  if (currentPlayer) {
    currentPlayer.pause();
    currentPlayer.remove();
    currentPlayer = null;
  }
}
```

`pause()` stopper lyden, og `remove()` frigør afspilleren fra hukommelsen. Den færdige løsning har også en lille `speechRequestId`, så et OpenAI-svar, som stadig er på vej, ikke begynder at spille **efter** brugeren har slået oplæsning fra. Se [Voice.js](../12_Tale_til_tekst_code/services/Voice.js).

### 2.3 Tekst til tale

OpenAI returnerer rå mp3-data. Vi skriver dem til en fil i `Paths.cache` og giver filens `uri` til `createAudioPlayer`.

```javascript
export async function speakText(text) {
  if (!text) return;

  try {
    const response = await fetch(SPEECH_URL, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${OPENAI_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "gpt-4o-mini-tts",
        voice: "???",
        input: text,
        instructions: "???",
        response_format: "mp3",
      }),
    });

    if (!response.ok) {
      console.error("Fejl fra OpenAI (oplæsning):", await response.text());
      return;
    }

    const bytes = new Uint8Array(await response.arrayBuffer());
    const file = new File(Paths.cache, `svar-${Date.now()}.mp3`);
    file.create();
    file.write(bytes);

    stopSpeaking();
    await setAudioModeAsync({
      playsInSilentMode: true,
      allowsRecording: false,
    });

    currentPlayer = createAudioPlayer(file.uri);
    currentPlayer.play();
  } catch (error) {
    console.error("Fejl ved oplæsning:", error);
  }
}
```

Udfyld begge `???`:

1. Stemmen skal være `coral`.
2. Skriv en instruktion, der beder stemmen tale **dansk**, i et **roligt tempo** og med en **venlig tone**.

`playsInSilentMode: true` gør, at iPhone kan spille svaret i lydløs tilstand. `allowsRecording: false` skifter væk fra optagetilstand, så afspilningen ikke bliver meget svag på iPhone.

**Tjek:** Har jeres `Voice.js` nu tre eksporterede funktioner: `transcribeAudio`, `speakText` og `stopSpeaking`?

---

## 3. Optag lyd i ChatScreen.js

Åbn jeres eksisterende `screens/ChatScreen.js`. Behold `onSend`, `handleIncomingMessage`, `respondWithBotMessage`, `getBardResp`, `renderBubble`, `renderInputToolbar` og `renderSend`. I tilføjer kun lydfunktionerne omkring dem.

### 3.1 Imports og state

Tilføj til imports. Hvis `useState`, `useRef` og `View` allerede er importeret, udvider I blot de eksisterende linjer.

```javascript
import { Pressable, Text, StyleSheet } from "react-native";
import Ionicons from "@expo/vector-icons/Ionicons";
import {
  useAudioRecorder,
  RecordingPresets,
  AudioModule,
  setAudioModeAsync,
} from "expo-audio";
import { transcribeAudio, speakText, stopSpeaking } from "../services/Voice";
```

Inde i `ChatScreen` opretter I optageren og en state til knappen:

```javascript
const recorder = useAudioRecorder(RecordingPresets.HIGH_QUALITY);
const [isRecording, setIsRecording] = useState(false);
```

Optagelsen gemmes som en lokal fil. Efter `await recorder.stop()` finder I dens sti i `recorder.uri`. `HIGH_QUALITY` giver et format, som Whisper kan modtage.

### 3.2 Start optagelsen

Brugeren skal give mikrofontilladelse. Når der er givet lov, stopper I eventuel oplæsning, skifter lydtilstand, forbereder optageren og starter den.

```javascript
const startRecording = async () => {
  try {
    const permission = await AudioModule.requestRecordingPermissionsAsync();
    if (!permission.???) {
      respondWithBotMessage(
        "Jeg har brug for adgang til mikrofonen for at kunne høre dig."
      );
      return;
    }

    stopSpeaking();
    await setAudioModeAsync({ playsInSilentMode: true, allowsRecording: ??? });
    await recorder.prepareToRecordAsync();
    recorder.record();
    setIsRecording(???);
  } catch (error) {
    console.error("Fejl ved optagelse:", error);
    respondWithBotMessage("Jeg kunne ikke starte optagelsen. Prøv igen.");
  }
};
```

Udfyld `???`: Hvad hedder tilladelsens boolean-felt? Skal `allowsRecording` være `true` eller `false` under optagelse? Hvad skal `isRecording` sættes til, så knappen skifter ikon?

### 3.3 Stop optagelsen og send beskeden

Efter stop viser I loading, mens Whisper arbejder. En succesfuld transskription skal laves til et almindeligt GiftedChat-beskedobjekt. Brugeren har `_id: 1` i øvelse 11; botten har `_id: 2`.

```javascript
const stopRecording = async () => {
  await recorder.stop();
  setIsRecording(false);

  setLoading(true);
  const text = await ???(recorder.uri);
  setLoading(false);

  if (!text) {
    respondWithBotMessage("Jeg kunne ikke høre, hvad du sagde. Prøv igen.");
    return;
  }

  const voiceMessage = {
    _id: Math.random() * (9999999 - 1),
    text,
    createdAt: Date.now(),
    user: { _id: ??? },
  };

  onSend([voiceMessage]);
};
```

Udfyld `???`: Hvilken funktion fra `Voice.js` laver lyd til tekst? Hvad er brugerens id i `<GiftedChat user={{ _id: 1 }}>`?

**Hvorfor `onSend`?** Den eksisterende `onSend` viser brugerbeskeden og kalder `handleIncomingMessage`. Derfra kører `runGuardrails`, før `getBardResp` kontakter tutor-modellen. En direkte kald til `getBardResp` her ville springe guardrails over.

Se den [færdige ChatScreen.js](../12_Tale_til_tekst_code/screens/ChatScreen.js) for `try`/`catch` og en lille lås, som forhindrer to hurtige tryk i at starte flere optagelser eller transskriptioner samtidig.

### 3.4 Mikrofonknappen

GiftedChat viser ekstra handlinger ved tekstfeltet gennem `renderActions`. Lav en funktion i `ChatScreen`:

```javascript
const renderMicButton = () => (
  <Pressable
    onPress={isRecording ? stopRecording : startRecording}
    style={styles.micButton}
    accessibilityRole="button"
    accessibilityLabel={isRecording ? "Stop optagelse" : "Start optagelse"}
  >
    <Ionicons
      name={isRecording ? "stop-circle" : "mic"}
      size={28}
      color={isRecording ? "#dc2626" : "#2563eb"}
    />
  </Pressable>
);
```

Føj `renderActions={renderMicButton}` til jeres eksisterende `<GiftedChat>`. Behold dens øvrige props, især `onSend`, `user`, `renderBubble`, `renderInputToolbar` og `renderSend`.

**Test nu:** Tryk på den blå mikrofon. Bliver den til en rød stopknap? Sig en hel dansk sætning og tryk stop. Bliver transskriptionen vist på højre side som jeres besked?

---

## 4. Læs tutorens svar højt

Vi vil kunne slå oplæsning til og fra uden at ændre på den eksisterende chatlogik. `respondWithBotMessage` bruges allerede til både AI-svar og guardrail-afvisninger. Derfor er det det rigtige sted at starte oplæsningen.

### 4.1 Hvorfor både state og ref?

I øvelse 11 er `onSend` en `useCallback(..., [])`. Den laves kun én gang og kan derfor se gamle state-værdier gennem funktionskæden. Brug `voiceOn` til at opdatere knappen på skærmen og `voiceOnRef.current` til at afgøre, om der skal afspilles lyd.

```javascript
const [voiceOn, setVoiceOn] = useState(true);
const voiceOnRef = useRef(true);

const toggleVoice = () => {
  const next = !voiceOnRef.current;
  voiceOnRef.current = next;
  setVoiceOn(next);
  if (!next) stopSpeaking();
};
```

Når brugeren slår oplæsning fra, afbrydes en igangværende afspilning med det samme.

### 4.2 Tilføj oplæsning i respondWithBotMessage

Behold den eksisterende kode, der bygger bot-beskeden og kalder `setMessages`. Tilføj bagefter:

```javascript
if (voiceOnRef.current) {
  speakText(text);
}
```

Brug **ref'en**, ikke `voiceOn`. Kald `speakText` uden `await`, så tekstboblen vises straks, mens lydfilen bliver hentet.

### 4.3 Vis oplæsningsknappen

Placér denne knap over `<GiftedChat>` inde i det eksisterende `View`:

```javascript
<Pressable onPress={toggleVoice} style={styles.voiceToggle}>
  <Ionicons
    name={voiceOn ? "volume-high" : "volume-mute"}
    size={20}
    color="#2563eb"
  />
  <Text>{voiceOn ? "Oplæsning til" : "Oplæsning fra"}</Text>
</Pressable>
```

Tilføj disse styles nederst i `ChatScreen.js`:

```javascript
const styles = StyleSheet.create({
  micButton: {
    width: 40,
    height: 40,
    marginLeft: 6,
    marginBottom: 4,
    alignItems: "center",
    justifyContent: "center",
  },
  voiceToggle: {
    flexDirection: "row",
    alignItems: "center",
    alignSelf: "flex-end",
    gap: 6,
    padding: 10,
  },
});
```

**Test nu:** Stil et spørgsmål, som tutoren gerne må besvare. Kommer tekstboblen frem med det samme? Bliver svaret læst højt? Slå oplæsning fra, og stil et nyt spørgsmål. Er der stille?

---

## 5. Test hele forløbet

Brug en fysisk telefon med Expo Go til mikrofontesten. Simulatoren er mindre egnet til optagelse. Test hvert trin for sig, så I ved, hvor en eventuel fejl ligger.

| Situation | Forventet resultat |
| --- | --- |
| Skriv en normal besked | Chatten virker som i øvelse 11 |
| Sig en dansk sætning | Den vises som en brugerbesked |
| Sig noget uden for emnet | Den eksisterende emne-guardrail afviser det |
| Sig en meget lang besked | Længde-guardrailen afviser teksten over 500 tegn |
| Afvis mikrofontilladelse | Appen viser en forståelig besked |
| Start optagelse under oplæsning | Oplæsningen stopper |
| Slå oplæsning fra | Næste bot-svar vises uden lyd |
| Brug iPhone i lydløs tilstand | Oplæsning kan stadig høres |

### Typiske fejl

| Problem | Kig her |
| --- | --- |
| `401` fra OpenAI | Kontroller `.env`, og genstart Expo med `npx expo start -c` |
| Intet optages | Kontroller mikrofontilladelse i telefonens indstillinger |
| Forkert sprog | Kontroller `language: "da"` og tal tydeligt |
| Lyd er svag på iPhone | Kontroller `allowsRecording: false` før afspilning |
| Talebesked står som bot-besked | Kontroller `user._id: 1` i `voiceMessage` |
| En gammel lyd starter efter tryk på Fra | Brug `speechRequestId`-sikringen fra facit i `Voice.js` |

Se også fejlene i den terminal, hvor Expo kører. `Voice.js` logger fejl fra begge lyd-endpoints med `console.error`.

---

## Tænk over

1. **Pris:** Transskription og oplæsning koster pr. brug. Find de aktuelle priser. Hvad koster det, hvis 1.000 brugere hver taler med tutoren i 10 minutter om dagen?
2. **Guardrails:** Længde-guardrailen tillader 500 tegn. Hvor mange sekunders tale svarer det cirka til? Transskriptionen er allerede betalt, før teksten når `runGuardrails`. Kunne I sætte en grænse for optagelsens varighed?
3. **Privatliv:** Brugerens stemme sendes til OpenAI. Hvad skal brugeren vide, før optagelsen starter? Ville det ændre jeres valg, hvis appen var til børn?
4. **Tilgængelighed:** Telefonen har ofte egen diktering og oplæsning. Hvornår giver OpenAI-lyd en bedre oplevelse for netop jeres tutor?

## Ekstraopgaver

- Vis teksten `Lytter...`, mens `isRecording` er `true`.
- Begræns længden på en optagelse og forklar brugeren hvorfor.
- Tilføj fagord som `prompt` ved transskription. Bliver de stavet mere korrekt?
- Afprøv en anden stemme og justér `instructions`, så stemmen passer til jeres tutors personlighed.

---

## Filcheckliste

- [ ] `services/Voice.js` med `transcribeAudio`, `speakText` og `stopSpeaking`
- [ ] `screens/ChatScreen.js` med optager, mikrofonknap og oplæsningsknap
- [ ] Talebeskeder går gennem `onSend` og eksisterende guardrails
- [ ] `app.json` har dansk mikrofontekst
- [ ] `.env` er fortsat ignoreret af Git

Den [komplette kode](../12_Tale_til_tekst_code/) viser alle trinene samlet, inklusive fejlhåndtering og sikring mod hurtige dobbeltryk.
