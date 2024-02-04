yarn add @whiskeysockets/baileys

yarn add github:WhiskeySockets/Baileys
import makeWASocket from '@whiskeysockets/baileys'
import makeWASocket, { DisconnectReason } from '@whiskeysockets/baileys'
import { Boom } from '@hapi/boom'

async function connectToWhatsApp () {
    const sock = makeWASocket({
        // can provide additional config here
        printQRInTerminal: true
    })
    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update
        if(connection === 'close') {
            const shouldReconnect = (lastDisconnect.error as Boom)?.output?.statusCode !== DisconnectReason.loggedOut
            console.log('connection closed due to ', lastDisconnect.error, ', reconnecting ', shouldReconnect)
            // reconnect if not logged out
            if(shouldReconnect) {
                connectToWhatsApp()
            }
        } else if(connection === 'open') {
            console.log('opened connection')
        }
    })
    sock.ev.on('messages.upsert', m => {
        console.log(JSON.stringify(m, undefined, 2))

        console.log('replying to', m.messages[0].key.remoteJid)
        await sock.sendMessage(m.messages[0].key.remoteJid!, { text: 'Hello there!' })
    })
}
// run in main file
connectToWhatsApp()
type SocketConfig = {
    /** the WS url to connect to WA */
    waWebSocketUrl: string | URL
    /** Fails the connection if the socket times out in this interval */
	connectTimeoutMs: number
    /** Default timeout for queries, undefined for no timeout */
    defaultQueryTimeoutMs: number | undefined
    /** ping-pong interval for WS connection */
    keepAliveIntervalMs: number
    /** proxy agent */
	agent?: Agent
    /** pino logger */
	logger: Logger
    /** version to connect with */
    version: WAVersion
    /** override browser config */
	browser: WABrowserDescription
	/** agent used for fetch requests -- uploading/downloading media */
	fetchAgent?: Agent
    /** should the QR be printed in the terminal */
    printQRInTerminal: boolean
    /** should events be emitted for actions done by this socket connection */
    emitOwnEvents: boolean
    /** provide a cache to store media, so does not have to be re-uploaded */
    mediaCache?: NodeCache
    /** custom upload hosts to upload media to */
    customUploadHosts: MediaConnInfo['hosts']
    /** time to wait between sending new retry requests */
    retryRequestDelayMs: number
    /** max msg retry count */
    maxMsgRetryCount: number
    /** time to wait for the generation of the next QR in ms */
    qrTimeout?: number;
    /** provide an auth state object to maintain the auth state */
    auth: AuthenticationState
    /** manage history processing with this control; by default will sync up everything */
    shouldSyncHistoryMessage: (msg: proto.Message.IHistorySyncNotification) => boolean
    /** transaction capability options for SignalKeyStore */
    transactionOpts: TransactionCapabilityOptions
    /** provide a cache to store a user's device list */
    userDevicesCache?: NodeCache
    /** marks the client as online whenever the socket successfully connects */
    markOnlineOnConnect: boolean
    /**
     * map to store the retry counts for failed messages;
     * used to determine whether to retry a message or not */
    msgRetryCounterMap?: MessageRetryMap
    /** width for link preview images */
    linkPreviewImageThumbnailWidth: number
    /** Should Baileys ask the phone for full history, will be received async */
    syncFullHistory: boolean
    /** Should baileys fire init queries automatically, default true */
    fireInitQueries: boolean
    /**
     * generate a high quality link preview,
     * entails uploading the jpegThumbnail to WA
     * */
    generateHighQualityLinkPreview: boolean

    /** options for axios */
    options: AxiosRequestConfig<any>
    /**
     * fetch a message from your store
     * implement this so that messages failed to send (solves the "this message can take a while" issue) can be retried
     * */
    getMessage: (key: proto.IMessageKey) => Promise<proto.IMessage | undefined>
}
const conn = makeWASocket({
    ...otherOpts,
    // can use Windows, Ubuntu here too
    browser: Browsers.macOS('Desktop'),
    syncFullHistory: true
})
import makeWASocket, { BufferJSON, useMultiFileAuthState } from '@whiskeysockets/baileys'
import * as fs from 'fs'

// utility function to help save the auth state in a single folder
// this function serves as a good guide to help write auth & key states for SQL/no-SQL databases, which I would recommend in any production grade system
const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')
// will use the given state to connect
// so if valid credentials are available -- it'll connect without QR
const conn = makeWASocket({ auth: state }) 
// this will be called as soon as the credentials are updated
conn.ev.on ('creds.update', saveCreds)type ConnectionState = {
	/** connection is now open, connecting or closed */
	connection: WAConnectionState
	/** the error that caused the connection to close */
	lastDisconnect?: {
		error: Error
		date: Date
	}
	/** is this a new login */
	isNewLogin?: boolean
	/** the current QR code */
	qr?: string
	/** has the device received all pending notifications while it was offline */
	receivedPendingNotifications?: boolean 
}
export type BaileysEventMap = {
    /** connection state has been updated -- WS closed, opened, connecting etc. */
	'connection.update': Partial<ConnectionState>
    /** credentials updated -- some metadata, keys or something */
    'creds.update': Partial<AuthenticationCreds>
    /** history sync, everything is reverse chronologically sorted */
    'messaging-history.set': {
        chats: Chat[]
        contacts: Contact[]
        messages: WAMessage[]
        isLatest: boolean
    }
    /** upsert chats */
    'chats.upsert': Chat[]
    /** update the given chats */
    'chats.update': Partial<Chat>[]
    /** delete chats with given ID */
    'chats.delete': string[]
    'labels.association': LabelAssociation
    'labels.edit': Label
    /** presence of contact in a chat updated */
    'presence.update': { id: string, presences: { [participant: string]: PresenceData } }

    'contacts.upsert': Contact[]
    'contacts.update': Partial<Contact>[]

    'messages.delete': { keys: WAMessageKey[] } | { jid: string, all: true }
    'messages.update': WAMessageUpdate[]
    'messages.media-update': { key: WAMessageKey, media?: { ciphertext: Uint8Array, iv: Uint8Array }, error?: Boom }[]
    /**
     * add/update the given messages. If they were received while the connection was online,
     * the update will have type: "notify"
     *  */
    'messages.upsert': { messages: WAMessage[], type: MessageUpsertType }
    /** message was reacted to. If reaction was removed -- then "reaction.text" will be falsey */
    'messages.reaction': { key: WAMessageKey, reaction: proto.IReaction }[]

    'message-receipt.update': MessageUserReceiptUpdate[]

    'groups.upsert': GroupMetadata[]
    'groups.update': Partial<GroupMetadata>[]
    /** apply an action to participants in a group */
    'group-participants.update': { id: string, participants: string[], action: ParticipantAction }

    'blocklist.set': { blocklist: string[] }
    'blocklist.update': { blocklist: string[], type: 'add' | 'remove' }
    /** Receive an update on a call, including when the call was received, rejected, accepted */
    'call': WACallEvent[]
}
const sock = makeWASocket()
sock.ev.on('messages.upsert', ({ messages }) => {
    console.log('got messages', messages)
})
import makeWASocket, { makeInMemoryStore } from '@whiskeysockets/baileys'
// the store maintains the data of the WA connection in memory
// can be written out to a file & read from it
const store = makeInMemoryStore({ })
// can be read from a file
store.readFromFile('./baileys_store.json')
// saves the state to a file every 10s
setInterval(() => {
    store.writeToFile('./baileys_store.json')
}, 10_000)

const sock = makeWASocket({ })
// will listen from this socket
// the store can listen from a new socket once the current socket outlives its lifetime
store.bind(sock.ev)

sock.ev.on('chats.set', () => {
    // can use "store.chats" however you want, even after the socket dies out
    // "chats" => a KeyedDB instance
    console.log('got chats', store.chats.all())
})

sock.ev.on('contacts.set', () => {
    console.log('got contacts', Object.values(store.contacts))
})
import { MessageType, MessageOptions, Mimetype } from '@whiskeysockets/baileys'

const id = 'abcd@s.whatsapp.net' // the WhatsApp ID 
// send a simple text!
const sentMsg  = await sock.sendMessage(id, { text: 'oh hello there' })
// send a reply messagge
const sentMsg  = await sock.sendMessage(id, { text: 'oh hello there' }, { quoted: message })
// send a mentions message
const sentMsg  = await sock.sendMessage(id, { text: '@12345678901', mentions: ['12345678901@s.whatsapp.net'] })
// send a location!
const sentMsg  = await sock.sendMessage(
    id, 
    { location: { degreesLatitude: 24.121231, degreesLongitude: 55.1121221 } }
)
// send a contact!
const vcard = 'BEGIN:VCARD\n' // metadata of the contact card
            + 'VERSION:3.0\n' 
            + 'FN:Jeff Singh\n' // full name
            + 'ORG:Ashoka Uni;\n' // the organization of the contact
            + 'TEL;type=CELL;type=VOICE;waid=911234567890:+91 12345 67890\n' // WhatsApp ID + phone number
            + 'END:VCARD'
const sentMsg  = await sock.sendMessage(
    id,
    { 
        contacts: { 
            displayName: 'Jeff', 
            contacts: [{ vcard }] 
        }
    }
)

const reactionMessage = {
    react: {
        text: "💖", // use an empty string to remove the reaction
        key: message.key
    }
}

const sendMsg = await sock.sendMessage(id, reactionMessage)
// send a link
const sentMsg  = await sock.sendMessage(id, { text: 'Hi, this was sent using https://github.com/adiwajshing/baileys' })
import { MessageType, MessageOptions, Mimetype } from '@whiskeysockets/baileys'
// Sending gifs
await sock.sendMessage(
    id, 
    { 
        video: fs.readFileSync("Media/ma_gif.mp4"), 
        caption: "hello!",
        gifPlayback: true
    }
)

await sock.sendMessage(
    id, 
    { 
        video: "./Media/ma_gif.mp4", 
        caption: "hello!",
        gifPlayback: true
    }
)

// send an audio file
await sock.sendMessage(
    id, 
    { audio: { url: "./Media/audio.mp3" }, mimetype: 'audio/mp4' }
    { url: "Media/audio.mp3" }, // can send mp3, mp4, & ogg
)
const info: MessageOptions = {
    quoted: quotedMessage, // the message you want to quote
    contextInfo: { forwardingScore: 2, isForwarded: true }, // some random context info (can show a forwarded message with this too)
    timestamp: Date(), // optional, if you want to manually set the timestamp of the message
    caption: "hello there!", // (for media messages) the caption to send with the media (cannot be sent with stickers though)
    jpegThumbnail: "23GD#4/==", /*  (for location & media messages) has to be a base 64 encoded JPEG if you want to send a custom thumb, 
                                or set to null if you don't want to send a thumbnail.
                                Do not enter this field if you want to automatically generate a thumb
                            */
    mimetype: Mimetype.pdf, /* (for media messages) specify the type of media (optional for all media types except documents),
                                import {Mimetype} from '@whiskeysockets/baileys'
                            */
    fileName: 'somefile.pdf', // (for media messages) file name for the media
    /* will send audio messages as voice notes, if set to true */
    ptt: true,
    /** Should it send as a disappearing messages. 
     * By default 'chat' -- which follows the setting of the chat */
    ephemeralExpiration: WA_DEFAULT_EPHEMERAL
}
const msg = getMessageFromStore('455@s.whatsapp.net', 'HSJHJWH7323HSJSJ') // implement this on your end
await sock.sendMessage('1234@s.whatsapp.net', { forward: msg }) // WA forward the message!
const key = {
    remoteJid: '1234-123@g.us',
    id: 'AHASHH123123AHGA', // id of the message you want to read
    participant: '912121232@s.whatsapp.net' // the ID of the user that sent the  message (undefined for individual chats)
}
// pass to readMessages function
// can pass multiple keys to read multiple messages as well
await sock.readMessages([key])
type WAPresence = 'unavailable' | 'available' | 'composing' | 'recording' | 'paused'
import { writeFile } from 'fs/promises'
import { downloadMediaMessage } from '@whiskeysockets/baileys'

sock.ev.on('messages.upsert', async ({ messages }) => {
    const m = messages[0]

    if (!m.message) return // if there is no text or media message
    const messageType = Object.keys (m.message)[0]// get what type of message it is -- text, image, video
    // if the message is an image
    if (messageType === 'imageMessage') {
        // download the message
        const buffer = await downloadMediaMessage(
            m,
            'buffer',
            { },
            { 
                logger,
                // pass this so that baileys can request a reupload of media
                // that has been deleted
                reuploadRequest: sock.updateMediaMessage
            }
        )
        // save to file
        await writeFile('./my-download.jpeg', buffer)
    }
}const updatedMediaMsg = await sock.updateMediaMessage(msg)const jid = '1234@s.whatsapp.net' // can also be a group
const response = await sock.sendMessage(jid, { text: 'hello!' }) // send a message
// sends a message to delete the given message
// this deletes the message for everyone
await sock.sendMessage(jid, { delete: response.key })const jid = '1234@s.whatsapp.net'

await sock.sendMessage(jid, {
      text: 'updated text goes here',
      edit: response.key,
    });const lastMsgInChat = await getLastMessageInChat('123456@s.whatsapp.net') // implement this on your end
await sock.chatModify({ archive: true, lastMessages: [lastMsgInChat] }, '123456@s.whatsapp.net')// mute for 8 hours
await sock.chatModify({ mute: 8*60*60*1000 }, '123456@s.whatsapp.net', [])
// unmute
await sock.chatModify({ mute: null }, '123456@s.whatsapp.net', [])
const lastMsgInChat = await getLastMessageInChat('123456@s.whatsapp.net') // implement this on your end
await sock.chatModify({
  delete: true,
  lastMessages: [{ key: lastMsgInChat.key, messageTimestamp: lastMsgInChat.messageTimestamp }]
},
'123456@s.whatsapp.net')
const lastMsgInChat = await getLastMessageInChat('123456@s.whatsapp.net') // implement this on your end
// mark it unread
await sock.chatModify({ markRead: false, lastMessages: [lastMsgInChat] }, '123456@s.whatsapp.net')
await sock.chatModify(
  { clear: { messages: [{ id: 'ATWYHDNNWU81732J', fromMe: true, timestamp: "1654823909" }] } }, 
  '123456@s.whatsapp.net', 
  []
  )
await sock.chatModify({
  pin: true // or `false` to unpin
},
'123456@s.whatsapp.net')
await sock.chatModify({
star: {
	messages: [{ id: 'messageID', fromMe: true // or `false` }],
    	star: true // - true: Star Message; false: Unstar Message
}},'123456@s.whatsapp.net');
const jid = '1234@s.whatsapp.net' // can also be a group
// turn on disappearing messages
await sock.sendMessage(
    jid, 
    // this is 1 week in seconds -- how long you want messages to appear for
    { disappearingMessagesInChat: WA_DEFAULT_EPHEMERAL }
)
// will send as a disappearing message
await sock.sendMessage(jid, { text: 'hello' }, { ephemeralExpiration: WA_DEFAULT_EPHEMERAL })
// turn off disappearing messages
await sock.sendMessage(
    jid, 
    { disappearingMessagesInChat: false }
)


const id = '123456'
const [result] = await sock.onWhatsApp(id)
if (result.exists) console.log (`${id} exists on WhatsApp, as jid: ${result.jid}`)
const status = await sock.fetchStatus("xyz@s.whatsapp.net")
console.log("status: " + status)
// for low res picture
const ppUrl = await sock.profilePictureUrl("xyz@g.us")
console.log("download profile picture from: " + ppUrl)
// for high res picture
const ppUrl = await sock.profilePictureUrl("xyz@g.us", 'image')
const status = 'Hello World!'
await sock.updateProfileStatus(status)
const name = 'My name'
await sock.updateProfileName(name)
// the presence update is fetched and called here
sock.ev.on('presence.update', json => console.log(json))
// request updates for a chat
await sock.presenceSubscribe("xyz@s.whatsapp.net") 
const jid = '111234567890-1594482450@g.us' // can be your own too
await sock.updateProfilePicture(jid, { url: './new-profile-picture.jpeg' })
const jid = '111234567890-1594482450@g.us' // can be your own too
await sock.removeProfilePicture(jid)
await sock.updateBlockStatus("xyz@s.whatsapp.net", "block") // Block user
await sock.updateBlockStatus("xyz@s.whatsapp.net", "unblock") // Unblock user
const profile = await sock.getBusinessProfile("xyz@s.whatsapp.net")
console.log("business description: " + profile.description + ", category: " + profile.category)
// title & participants
const group = await sock.groupCreate("My Fab Group", ["1234@s.whatsapp.net", "4564@s.whatsapp.net"])
console.log ("created group with id: " + group.gid)
sock.sendMessage(group.id, { text: 'hello there' }) // say hello to everyone on the group
// id & people to add to the group (will throw error if it fails)
const response = await sock.groupParticipantsUpdate(
    "abcd-xyz@g.us", 
    ["abcd@s.whatsapp.net", "efgh@s.whatsapp.net"],
    "add" // replace this parameter with "remove", "demote" or "promote"
)
await sock.groupUpdateDescription("abcd-xyz@g.us", "New Description!")
await sock.groupUpdateSubject("abcd-xyz@g.us", "New Subject!")
// only allow admins to send messages
await sock.groupSettingUpdate("abcd-xyz@g.us", 'announcement')
// allow everyone to send messages
await sock.groupSettingUpdate("abcd-xyz@g.us", 'not_announcement')
// allow everyone to modify the group's settings -- like display picture etc.
await sock.groupSettingUpdate("abcd-xyz@g.us", 'unlocked')
// only allow admins to modify the group's settings
await sock.groupSettingUpdate("abcd-xyz@g.us", 'locked')
await sock.groupLeave("abcd-xyz@g.us") // (will throw error if it fails)
const code = await sock.groupInviteCode("abcd-xyz@g.us")
console.log("group code: " + code)
const code = await sock.groupRevokeInvite("abcd-xyz@g.us")
console.log("New group code: " + code)
const metadata = await sock.groupMetadata("abcd-xyz@g.us") 
console.log(metadata.id + ", title: " + metadata.subject + ", description: " + metadata.desc)
const response = await sock.groupAcceptInvite("xxx")
console.log("joined to: " + response)
const response = await sock.groupGetInviteInfo("xxx")
console.log("group information: " + response)
const response = await sock.groupAcceptInviteV4("abcd@s.whatsapp.net", groupInviteMessage)
console.log("joined to: " + response)
const response = await sock.groupRequestParticipantsList("abcd-xyz@g.us")
console.log(response)
const response = await sock.groupRequestParticipantsUpdate(
    "abcd-xyz@g.us", // id group,
    ["abcd@s.whatsapp.net", "efgh@s.whatsapp.net"],
    "approve" // replace this parameter with "reject" 
)
console.log(response)
const privacySettings = await sock.fetchPrivacySettings(true)
console.log("privacy settings: " + privacySettings)
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateLastSeenPrivacy(value)
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateProfilePicturePrivacy(value)
const value = 'all' // 'match_last_seen'
await sock.updateOnlinePrivacy(value)
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateStatusPrivacy(value)
const value = 'all' // 'none'
await sock.updateReadReceiptsPrivacy(value)
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateGroupsAddPrivacy(value)
const duration = 86400 // 604800 | 7776000 | 0 
await sock.updateDefaultDisappearingMode(duration)
sock.sendMessage(jid, {image: {url: url}, caption: caption}, {backgroundColor : backgroundColor, font : font, statusJidList: statusJidList, broadcast : true})
const bList = await sock.getBroadcastListInfo("1234@broadcast")
console.log (`list name: ${bList.name}, recps: ${bList.recipients}`)
const sock = makeWASocket({
    logger: P({ level: 'debug' }),
})
// for any message with tag 'edge_routing'
sock.ws.on(`CB:edge_routing`, (node: BinaryNode) => { })
// for any message with tag 'edge_routing' and id attribute = abcd
sock.ws.on(`CB:edge_routing,id:abcd`, (node: BinaryNode) => { })
// for any message with tag 'edge_routing', id attribute = abcd & first content node routing_info
sock.ws.on(`CB:edge_routing,id:abcd,routing_info`, (node: BinaryNode) => { })
