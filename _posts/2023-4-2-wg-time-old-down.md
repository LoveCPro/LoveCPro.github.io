---
layout: post
title:  wg源码解读之时间引起的隧道不能建立
subtitle:   TCP伪装成UDP报文的应用
date:   2023-4-2T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: Linux
catalog: true
tags:
    -  VPN开发
---
# 前言
唉，又是一个由现网故障所发现的问题。现网每天都会把客户端设备重启，而且还会修改设备时间，当时还没找到我，现网发现隧道不通，就重新配置了业务。后来技术支持找到我说业务不通，还没说修改时间的事，我也不知道时间会导致严重问题，我就在他提供的信息上一通排查。

# 源码简单解析
请结合《wireguard介绍》看。
- wg版本：20190702
## 数据结构
- 消息类型
```c
enum message_type {
	MESSAGE_INVALID = 0,
	MESSAGE_HANDSHAKE_INITIATION = 1,   //握手请求
	MESSAGE_HANDSHAKE_RESPONSE = 2,     //握手响应
	MESSAGE_HANDSHAKE_COOKIE = 3,       //握手cookie
	MESSAGE_DATA = 4
};
```
- 握手请求结构体
```c
struct message_handshake_initiation {
	struct message_header header;
	__le32 sender_index;
	u8 unencrypted_ephemeral[NOISE_PUBLIC_KEY_LEN]; //发送方为这次握手临时生成的公钥（未加密，用于 ECDH）
	u8 encrypted_static[noise_encrypted_len(NOISE_PUBLIC_KEY_LEN)];//用对端公钥和临时生成的私钥 ECDH 出的临时密钥 key1 对称加密对方的公钥
	u8 encrypted_timestamp[noise_encrypted_len(NOISE_TIMESTAMP_LEN)];//用对端公钥和自己的私钥 ECDH 出 key2，key2 混淆进 key1，来加密当前的时间戳
	struct message_macs macs;       //对端公钥加上整个报文内容后的哈希
};

```
- 握手响应结构体
```c
struct message_handshake_response {
	struct message_header header;
	__le32 sender_index;
	__le32 receiver_index;
	u8 unencrypted_ephemeral[NOISE_PUBLIC_KEY_LEN]; //接收方为这次握手临时生成的公钥（未加密，用于 ECDH）
	u8 encrypted_nothing[noise_encrypted_len(0)];
	struct message_macs macs;//对端公钥加上整个报文内容后的哈希
};

```
- 握手cookie结构体
```c
struct message_handshake_cookie {
	struct message_header header;
	__le32 receiver_index;
	u8 nonce[COOKIE_NONCE_LEN];//随机数
	u8 encrypted_cookie[noise_encrypted_len(COOKIE_LEN)];//AEAD加密后的cookie
};

```
- 数据结构体
```c
struct message_data {
	struct message_header header;
	__le32 key_idx;
	__le64 counter;
	u8 encrypted_data[];//AEAD加密后的inner IP pkt
};
```
## 握手交互
- 发送握手请求
```c
static void wg_packet_send_handshake_initiation(struct wg_peer *peer)
{
	struct message_handshake_initiation packet;

	if (!wg_birthdate_has_expired(atomic64_read(&peer->last_sent_handshake),
				      REKEY_TIMEOUT))
		return; /* This function is rate limited. */

	atomic64_set(&peer->last_sent_handshake, ktime_get_coarse_boottime_ns());
	net_dbg_ratelimited("%s: Sending handshake initiation to peer %llu (%pISpfsc)\n",
			    peer->device->dev->name, peer->internal_id,
			    &peer->endpoint.addr);

    // 构造一个握手发起消息，赋值macs之前内容
	if (wg_noise_handshake_create_initiation(&packet, &peer->handshake)) {
        //赋值macs.mac1和macs.mac2
        // macs.mac1为 对端公钥为参数的blake2s哈希值, mac2为0或者上一个响应的cookie的哈希值
		wg_cookie_add_mac_to_packet(&packet, sizeof(packet), peer);
		wg_timers_any_authenticated_packet_traversal(peer);
		wg_timers_any_authenticated_packet_sent(peer);
		atomic64_set(&peer->last_sent_handshake,
			     ktime_get_coarse_boottime_ns());
		wg_socket_send_buffer_to_peer(peer, &packet, sizeof(packet),
					      HANDSHAKE_DSCP);
		wg_timers_handshake_initiated(peer);
	}
}

bool wg_noise_handshake_create_initiation(struct message_handshake_initiation *dst,
				     struct noise_handshake *handshake)
{
	u8 timestamp[NOISE_TIMESTAMP_LEN];
	u8 key[NOISE_SYMMETRIC_KEY_LEN];
	bool ret = false;

	/* We need to wait for crng _before_ taking any locks, since
	 * curve25519_generate_secret uses get_random_bytes_wait.
	 */
	wait_for_random_bytes();

	down_read(&handshake->static_identity->lock);
	down_write(&handshake->lock);

	if (unlikely(!handshake->static_identity->has_identity))
		goto out;

    // type为0x01，发起消息
	dst->header.type = cpu_to_le32(MESSAGE_HANDSHAKE_INITIATION);

    //使用对端公钥初始化握手chaining_key和hash
    //chaining_key初始值为Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s的blake2哈希值
    //hash是与chaining_key和对端公钥有关的blake2哈希值
	handshake_init(handshake->chaining_key, handshake->hash,
		       handshake->remote_static);

	/* e */
    //生成临时私钥
	curve25519_generate_secret(handshake->ephemeral_private);
    //临时私钥生成临时公钥
	if (!curve25519_generate_public(dst->unencrypted_ephemeral,
					handshake->ephemeral_private))
		goto out;
    // 临时公钥参与哈希计算更新hash和chaining_key
	message_ephemeral(dst->unencrypted_ephemeral,
			  dst->unencrypted_ephemeral, handshake->chaining_key,
			  handshake->hash);

	/* es */
    // 临时私钥与对端公钥点乘(DH)计算出一个值，该值参与哈希计算更新chaining_key和key
	if (!mix_dh(handshake->chaining_key, key, handshake->ephemeral_private,
		    handshake->remote_static))
		goto out;

	/* s */
    // 本地公钥用上述key AEAD加密，hash随之更新
	message_encrypt(dst->encrypted_static,
			handshake->static_identity->static_public,
			NOISE_PUBLIC_KEY_LEN, key, handshake->hash);

	/* ss */
    // 预静态密钥和chaining_key哈希更新key和chaining_key(kdf算法)
	if (!mix_precomputed_dh(handshake->chaining_key, key,
				handshake->precomputed_static_static))
		goto out;

	/* {t} */
	tai64n_now(timestamp);
    // 时间戳用上述key加密，hash随之更新
	message_encrypt(dst->encrypted_timestamp, timestamp,
			NOISE_TIMESTAMP_LEN, key, handshake->hash);
	// 握手信息插入哈希表
	dst->sender_index = wg_index_hashtable_insert(
		handshake->entry.peer->device->index_hashtable,
		&handshake->entry);

	handshake->state = HANDSHAKE_CREATED_INITIATION;
	ret = true;

out:
	up_write(&handshake->lock);
	up_read(&handshake->static_identity->lock);
	memzero_explicit(key, NOISE_SYMMETRIC_KEY_LEN);
	return ret;
}

```
- 服务端处理握手请求
```c
struct wg_peer *
wg_noise_handshake_consume_initiation(struct message_handshake_initiation *src,
				      struct wg_device *wg)
{
	struct wg_peer *peer = NULL, *ret_peer = NULL;
	struct noise_handshake *handshake;
	bool replay_attack, flood_attack;
	u8 key[NOISE_SYMMETRIC_KEY_LEN];
	u8 chaining_key[NOISE_HASH_LEN];
	u8 hash[NOISE_HASH_LEN];
	u8 s[NOISE_PUBLIC_KEY_LEN];
	u8 e[NOISE_PUBLIC_KEY_LEN];
	u8 t[NOISE_TIMESTAMP_LEN];
	u64 initiation_consumption;

	down_read(&wg->static_identity.lock);
	if (unlikely(!wg->static_identity.has_identity))
		goto out;
	// 本地公钥初始化hash和chaining_key和hash
	handshake_init(chaining_key, hash, wg->static_identity.static_public);

	/* e */
    // 对端临时公钥参与哈希计算更新hash和chaining_key
	message_ephemeral(e, src->unencrypted_ephemeral, chaining_key, hash);

	/* es */
    // 本地私钥与对端临时公钥点乘(DH)计算出一个值，该值参与哈希计算更新chaining_key和key
	if (!mix_dh(chaining_key, key, wg->static_identity.static_private, e))
		goto out;

	/* s */
    // 用上述key AEAD解密对端公钥，hash随之更新
	if (!message_decrypt(s, src->encrypted_static,
			     sizeof(src->encrypted_static), key, hash))
		goto out;

	/* Lookup which peer we're actually talking to */
    // 对端公钥查寻peer
	peer = wg_pubkey_hashtable_lookup(wg->peer_hashtable, s);
	if (!peer)
		goto out;
	handshake = &peer->handshake;

	/* ss */
    // 预静态密钥和chaining_key哈希更新key和chaining_key(kdf算法)
	if (!mix_precomputed_dh(chaining_key, key,
				handshake->precomputed_static_static))
	    goto out;

	/* {t} */
    // 利用上述key hash AEAD解密时间戳
	if (!message_decrypt(t, src->encrypted_timestamp,
			     sizeof(src->encrypted_timestamp), key, hash))
		goto out;

	down_read(&handshake->lock);
    // 比较时间戳是否小于等于上次时间戳，检查重放攻击
	replay_attack = memcmp(t, handshake->latest_timestamp,
			       NOISE_TIMESTAMP_LEN) <= 0;
    // 计算请求时间间隔，检查洪水攻击
	flood_attack = (s64)handshake->last_initiation_consumption +
			       NSEC_PER_SEC / INITIATIONS_PER_SECOND >
		       (s64)ktime_get_coarse_boottime_ns();
	up_read(&handshake->lock);
	if (replay_attack || flood_attack)
		goto out;

	/* Success! Copy everything to peer */
	down_write(&handshake->lock);
    // 记录对端公钥
	memcpy(handshake->remote_ephemeral, e, NOISE_PUBLIC_KEY_LEN);
    // 更新请求时间
	if (memcmp(t, handshake->latest_timestamp, NOISE_TIMESTAMP_LEN) > 0)
		memcpy(handshake->latest_timestamp, t, NOISE_TIMESTAMP_LEN);
	// 记录hash和chaining_key
    memcpy(handshake->hash, hash, NOISE_HASH_LEN);
	memcpy(handshake->chaining_key, chaining_key, NOISE_HASH_LEN);
	handshake->remote_index = src->sender_index;
	initiation_consumption = ktime_get_coarse_boottime_ns();
	if ((s64)(handshake->last_initiation_consumption - initiation_consumption) < 0)
		handshake->last_initiation_consumption = initiation_consumption;
	handshake->state = HANDSHAKE_CONSUMED_INITIATION;
	up_write(&handshake->lock);
	ret_peer = peer;

out:
	memzero_explicit(key, NOISE_SYMMETRIC_KEY_LEN);
	memzero_explicit(hash, NOISE_HASH_LEN);
	memzero_explicit(chaining_key, NOISE_HASH_LEN);
	up_read(&wg->static_identity.lock);
	if (!ret_peer)
		wg_peer_put(peer);
	return ret_peer;
}

```
看到这儿其实我的问题已经解决了，看如下代码：
```c
	down_read(&handshake->lock);
    // 比较时间戳是否小于等于上次时间戳，检查重放攻击
	replay_attack = memcmp(t, handshake->latest_timestamp,
			       NOISE_TIMESTAMP_LEN) <= 0;
    // 计算请求时间间隔，检查洪水攻击
	flood_attack = (s64)handshake->last_initiation_consumption +
			       NSEC_PER_SEC / INITIATIONS_PER_SECOND >
		       (s64)ktime_get_coarse_boottime_ns();
```
>WireGuard的一个设计目标是避免在身份验证之前存储任何状态，并且不对未经身份验证的数据包发送任何响应。


>通过不允许未经身份验证的数据包影响任何状态，可以避免几类攻击。
>有可能以一种根本不需要动态内存分配的方式来实现WireGuard，即使对于经过身份验证的数据包也是如此。但是，此属性要求响应者收到的第一条消息就对发起者进行身份验证。 像这样在第一个数据包中进行身份验证可能会使响应者面临重放攻击的风险。攻击者可以重播初始握手消息，以诱使响应者重新生成其临时密钥，从而使合法发起者的会话无效（尽管不会影响任何消息的保密性或真实性）。为防止这种情况，在第一个消息中包含12字节的TAI64N [7]时间戳，并对其进行加密和身份验证。响应者跟踪每个对等方收到的最大时间戳，并丢弃包含小于或等于该时间戳的数据包。

详细说明见<https://cshihong.github.io/2020/10/11/WireGuard%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/>

- 服务端发送握手响应  
  接收握手请求之后，校验正确之后，响应者会发送握手响应消息，并计算出共享密钥。
```c
void wg_packet_send_handshake_response(struct wg_peer *peer)
{
	struct message_handshake_response packet;

	atomic64_set(&peer->last_sent_handshake, ktime_get_coarse_boottime_ns());
	net_dbg_ratelimited("%s: Sending handshake response to peer %llu (%pISpfsc)\n",
			    peer->device->dev->name, peer->internal_id,
			    &peer->endpoint.addr);
		//构造握手响应报文
	if (wg_noise_handshake_create_response(&packet, &peer->handshake)) {
        // 为报文中mac1, mac2赋值
		wg_cookie_add_mac_to_packet(&packet, sizeof(packet), peer);
		if (wg_noise_handshake_begin_session(&peer->handshake,
						     &peer->keypairs)) {
			wg_timers_session_derived(peer);
			wg_timers_any_authenticated_packet_traversal(peer);
			wg_timers_any_authenticated_packet_sent(peer);
			atomic64_set(&peer->last_sent_handshake,
				     ktime_get_coarse_boottime_ns());
			wg_socket_send_buffer_to_peer(peer, &packet,
						      sizeof(packet),
						      HANDSHAKE_DSCP);
		}
	}
}

bool wg_noise_handshake_create_response(struct message_handshake_response *dst,
					struct noise_handshake *handshake)
{
	u8 key[NOISE_SYMMETRIC_KEY_LEN];
	bool ret = false;

	/* We need to wait for crng _before_ taking any locks, since
	 * curve25519_generate_secret uses get_random_bytes_wait.
	 */
	wait_for_random_bytes();

	down_read(&handshake->static_identity->lock);
	down_write(&handshake->lock);

	if (handshake->state != HANDSHAKE_CONSUMED_INITIATION)
		goto out;

	dst->header.type = cpu_to_le32(MESSAGE_HANDSHAKE_RESPONSE);
	dst->receiver_index = handshake->remote_index;

	/* e */
    //生成临时私钥和临时公钥
	curve25519_generate_secret(handshake->ephemeral_private);
	if (!curve25519_generate_public(dst->unencrypted_ephemeral,
					handshake->ephemeral_private))
		goto out;
    // 临时公钥参与哈希计算更新hash和chaining_key
    // 此处的hash和chaining_key为解析请求包时更新的结果
	message_ephemeral(dst->unencrypted_ephemeral,
			  dst->unencrypted_ephemeral, handshake->chaining_key,
			  handshake->hash);

	/* ee */
    // 临时私钥与对端临时公钥点乘(DH)计算出一个值，该值参与哈希计算更新chaining_key和key
	if (!mix_dh(handshake->chaining_key, NULL, handshake->ephemeral_private,
		    handshake->remote_ephemeral))
		goto out;

	/* se */
    // 临时私钥与对端公钥点乘(DH)计算出一个值，该值参与哈希计算更新chaining_key和key
	if (!mix_dh(handshake->chaining_key, NULL, handshake->ephemeral_private,
		    handshake->remote_static))
		goto out;

	/* psk */
    // 利用预共享密钥kdf计算更新chaining_key、hash和key
	mix_psk(handshake->chaining_key, handshake->hash, key,
		handshake->preshared_key);

	/* {} */
    // 利用上述key和hash AEAD加密一段空内容, hash更新
	message_encrypt(dst->encrypted_nothing, NULL, 0, key, handshake->hash);

    // 将该握手信息插入索引哈希表
	dst->sender_index = wg_index_hashtable_insert(
		handshake->entry.peer->device->index_hashtable,
		&handshake->entry);

	handshake->state = HANDSHAKE_CREATED_RESPONSE;
	ret = true;

out:
	up_write(&handshake->lock);
	up_read(&handshake->static_identity->lock);
	memzero_explicit(key, NOISE_SYMMETRIC_KEY_LEN);
	return ret;
}
```
