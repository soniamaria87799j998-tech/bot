import asyncio
import os
import re
import time
from telethon import TelegramClient, events
from telethon.sessions import StringSession
from telegram import Bot
from datetime import datetime
from telethon.tl.types import PeerChannel, PeerChat, PeerUser
from telethon.errors import ChatForwardsRestrictedError
from collections import deque


class TelegramForwarder:
    SESSION_FILE = "telegram_session.txt"

    def __init__(self):
        # 配置部分
        self.USER_API_ID = 21393819
        self.USER_API_HASH = '8a5098a01d2956fecdb5f3e828a6a724'
        self.BOT_TOKEN = '8001729355:AAHOzoyi81vbwtkSWBR-GN3--xyse1sOkKc'
        self.KEYWORDS = ['点击验证后领取', '发送了一个红包', '小天•白金担保负责人 的红包', '请选择正确的验证码',
                         '⚠️ 同设备 / IP 禁止作弊 → 违规清零，无争议', '金貝招商444 （请直奔主题）@jbqb金貝担保发了',
                         '⚠️ 严禁一人多号或团队操作，违规取消资格！']
        self.TARGET_CHAT_IDS = [-1002949234138]

        # 货币金额阈值配置
        self.CURRENCY_THRESHOLDS = {
            'CNY': 5.0,  # 人民币
            'RMB': 5.0,  # 人民币
            'USD': 5.0,  # 美元
            'USDT': 5.0,  # USDT
            'EUR': 5.0,  # 欧元
            'GBP': 5.0,  # 英镑
            'JIBA': 10000.0,  # JIBA 货币
            'default': 10000.0  # 默认阈值
        }

        # 货币符号映射
        self.CURRENCY_SYMBOLS = {
            '¥': 'CNY',
            '￥': 'CNY',
            '元': 'CNY',
            '人民币': 'CNY',
            'RMB': 'CNY',
            '$': 'USD',
            'USD': 'USD',
            'USDT': 'USDT',
            '€': 'EUR',
            '£': 'GBP',
            'JIBA': 'JIBA'
        }

        # 初始化客户端
        self.session_string = self._load_session()
        self.user_client = TelegramClient(
            StringSession(self.session_string),
            self.USER_API_ID,
            self.USER_API_HASH
        )
        self.bot = Bot(token=self.BOT_TOKEN)

        # 消息处理队列和状态跟踪
        self.message_queue = deque()
        self.processing = False
        self.last_processed_time = 0
        self.message_counter = 0
        self.start_time = time.time()

    def _load_session(self):
        """加载已保存的会话"""
        if os.path.exists(self.SESSION_FILE):
            with open(self.SESSION_FILE, 'r') as f:
                return f.read().strip()
        return ''

    def _save_session(self):
        """保存当前会话"""
        if self.user_client.session:
            with open(self.SESSION_FILE, 'w') as f:
                f.write(self.user_client.session.save())

    async def _ensure_connection(self):
        """确保连接正常"""
        if not self.user_client.is_connected():
            await self.user_client.connect()

        if not await self.user_client.is_user_authorized():
            print("首次使用，请登录...")
            await self.user_client.start()
            self._save_session()
            print(f"会话已保存到 {self.SESSION_FILE}")

    async def _verify_chat_access(self, chat_id):
        """验证是否可以访问目标聊天"""
        try:
            await self.user_client.get_entity(chat_id)
            return True
        except Exception as e:
            print(f"无法访问聊天 {chat_id}: {str(e)}")
            return False

    async def forward_message(self, message):
        """转发消息实现 - 优化版：即使转发失败也发送信息链接"""
        try:
            await self._ensure_connection()

            chat = await message.get_chat()
            sender = await message.get_sender()

            # 获取聊天类型和名称
            if isinstance(chat, PeerChannel):
                chat_type = "频道"
                chat_title = getattr(chat, 'title', '未知频道')
            elif isinstance(chat, PeerChat):
                chat_type = "群组"
                chat_title = getattr(chat, 'title', '未知群组')
            elif isinstance(chat, PeerUser):
                chat_type = "私聊"
                chat_title = f"{getattr(sender, 'first_name', '')} {getattr(sender, 'last_name', '')}".strip()
            else:
                chat_type = "未知类型"
                chat_title = "未知聊天"

            sender_name = f"{getattr(sender, 'first_name', '')} {getattr(sender, 'last_name', '')}".strip()

            # 生成消息链接 - 这是关键部分
            message_link = "链接生成失败"
            try:
                if hasattr(chat, 'id'):
                    if isinstance(chat.id, int):
                        # 处理频道/超级群组ID
                        if hasattr(chat, 'username') and chat.username:
                            message_link = f"https://t.me/{chat.username}/{message.id}"
                        else:
                            # 私有频道/群组
                            channel_id = str(chat.id)
                            if channel_id.startswith('-100'):
                                channel_id = channel_id[4:]
                            message_link = f"https://t.me/c/{channel_id}/{message.id}"
                    else:
                        message_link = f"链接不可用(非标准ID: {chat.id})"
                else:
                    message_link = "链接不可用(无ID)"
            except Exception as e:
                print(f"生成消息链接失败: {str(e)}")
                message_link = "链接生成错误"

            # 提取消息预览文本
            message_text = message.text or ""
            # 清理文本：去除多余空格和换行
            message_text = re.sub(r'\s+', ' ', message_text).strip()
            # 智能截断
            if len(message_text) > 100:
                preview_text = message_text[:97] + "..."
            else:
                preview_text = message_text

            if not preview_text:
                preview_text = "无文本内容"

            # 为每个目标群组单独处理
            for target_chat_id in self.TARGET_CHAT_IDS:
                # 跳过无法访问的群组
                if not await self._verify_chat_access(target_chat_id):
                    continue

                forward_success = False
                forward_error = ""

                try:
                    # 尝试转发原始消息
                    await self.user_client.forward_messages(
                        entity=target_chat_id,
                        messages=message
                    )
                    forward_success = True
                except ChatForwardsRestrictedError:
                    forward_error = "目标群组禁止转发消息"
                except Exception as e:
                    forward_error = f"转发失败: {str(e)}"

                # 构造转发信息 - 包含转发状态和内容预览
                status_msg = "✅ 已成功转发原始信息" if forward_success else f"⚠️ {forward_error}"

                forward_info = (
                    f"🧧 新的红包提醒\n"
                    f"----------------------------\n"
                    f"类型: {chat_type}\n"
                    f"来源: {chat_title}\n"
                    f"发送者: {sender_name}\n"
                    f"状态: {status_msg}\n"
                    f"消息链接: {message_link}\n"
                    f"时间: {message.date.strftime('%Y-%m-%d %H:%M:%S')}\n"
                    f"----------------------------\n"
                    f"内容预览:\n{preview_text}\n"
                    f"----------------------------"
                )

                try:
                    # 发送信息消息 - 即使转发失败也发送
                    await self.user_client.send_message(
                        entity=target_chat_id,
                        message=forward_info,
                        link_preview=False
                    )
                    print(f"已发送关键词通知到群组 {target_chat_id}")
                except Exception as e:
                    print(f"发送信息消息到 {target_chat_id} 失败: {str(e)}")

            print(f"[{datetime.now()}] 已处理关键词消息 | 来源: {chat_title}")

        except Exception as e:
            print(f"消息处理失败: {str(e)}")

    def extract_currency_amount(self, text):
        """
        增强版货币金额提取函数
        支持格式: "总金额：100JIBA", "¥5.88", "10元", "5 USDT"
        """
        # 尝试匹配"总金额：100JIBA"格式
        total_match = re.search(r'总金额\s*[:：]\s*(\d+(?:\.\d+)?)\s*([A-Za-z]{2,})', text, re.IGNORECASE)
        if total_match:
            try:
                amount = float(total_match.group(1))
                currency = total_match.group(2).upper()
                return currency, amount
            except:
                pass

        # 尝试匹配"100JIBA"格式
        amount_match = re.search(r'(\d+(?:\.\d+)?)\s*([A-Za-z]{2,})', text, re.IGNORECASE)
        if amount_match:
            try:
                amount = float(amount_match.group(1))
                currency = amount_match.group(2).upper()
                return currency, amount
            except:
                pass

        # 尝试匹配"JIBA100"格式
        currency_first_match = re.search(r'([A-Za-z]{2,})\s*(\d+(?:\.\d+)?)', text, re.IGNORECASE)
        if currency_first_match:
            try:
                currency = currency_first_match.group(1).upper()
                amount = float(currency_first_match.group(2))
                return currency, amount
            except:
                pass

        # 尝试匹配标准货币格式
        standard_patterns = [
            # 格式: ¥5.88, $10, €20.5
            r'([¥￥$€£])\s*(\d+(?:\.\d+)?)',
            # 格式: 5.88元, 10人民币, 5 USDT
            r'(\d+(?:\.\d+)?)\s*(元|人民币|RMB|USD|USDT|€|£)',
            # 格式: 5.88元红包, 10人民币红包
            r'(\d+(?:\.\d+)?)\s*(元|人民币|RMB|USD|USDT|€|£)\s*红包',
            # 格式: 红包5.88元, 红包10人民币
            r'红包\s*(\d+(?:\.\d+)?)\s*(元|人民币|RMB|USD|USDT|€|£)',
            # 格式: ￥5.88, ¥10.00, $5.88
            r'([¥￥$€£])(\d+(?:\.\d+)?)',
            # 格式: 5.88, 10.00 (纯数字)
            r'(\d+(?:\.\d+)?)\s*(元|人民币|RMB|USD|USDT|€|£)?',
        ]

        for pattern in standard_patterns:
            match = re.search(pattern, text)
            if match:
                try:
                    # 提取金额和货币符号
                    if match.lastindex == 2:
                        amount = float(match.group(1) if match.group(1) is not None else match.group(2))
                        currency_symbol = match.group(2) if match.group(2) is not None else match.group(1)
                    else:
                        amount = float(match.group(1))
                        currency_symbol = match.group(2) if len(match.groups()) > 1 else None

                    # 将货币符号转换为标准货币代码
                    currency = None
                    if currency_symbol:
                        # 在符号映射中查找
                        for symbol, code in self.CURRENCY_SYMBOLS.items():
                            if symbol in currency_symbol:
                                currency = code
                                break

                    # 如果没有识别到货币，但金额提取成功，则使用默认货币
                    if currency is None and amount is not None:
                        # 检查文本中是否有货币提示
                        if any(keyword in text for keyword in ['$', 'USD', 'USDT']):
                            currency = 'USD'
                        elif any(keyword in text for keyword in ['€', 'EUR']):
                            currency = 'EUR'
                        elif any(keyword in text for keyword in ['£', 'GBP']):
                            currency = 'GBP'
                        elif any(keyword in text for keyword in ['¥', '￥', '元', '人民币', 'RMB']):
                            currency = 'CNY'
                        elif "JIBA" in text.upper():
                            currency = 'JIBA'
                        else:
                            currency = 'CNY'  # 默认人民币

                    return currency, amount
                except:
                    continue

        # 尝试提取纯数字金额
        amount_match = re.search(r'(\d+(?:\.\d+)?)', text)
        if amount_match:
            try:
                amount = float(amount_match.group(1))
                # 检查文本中是否有货币提示
                if "JIBA" in text.upper():
                    return "JIBA", amount
                else:
                    return "CNY", amount  # 默认人民币
            except:
                pass

        return None, None

    async def process_message_queue(self):
        """处理消息队列中的消息"""
        while True:
            if self.message_queue:
                # 获取队列中的第一条消息
                event = self.message_queue.popleft()

                # 处理消息
                try:
                    message = event.message
                    text = message.text or ''

                    # 记录处理开始时间
                    start_time = time.time()

                    # ===================== 红包过滤逻辑 =====================
                    # 检查消息是否包含"红包"关键词
                    if "红包" in text:
                        # 检查是否是专属红包（包含"专属"或@特定用户）
                        if "专属" in text or re.search(r'@\w+', text):
                            print(f"检测到专属红包，跳过处理: {text[:50]}...")
                            continue

                        # 提取货币类型和金额
                        currency, amount = self.extract_currency_amount(text)

                        if amount is not None:
                            # 获取该货币的阈值
                            if currency:
                                threshold = self.CURRENCY_THRESHOLDS.get(currency,
                                                     self.CURRENCY_THRESHOLDS.get('default', 5.0))
                            else:
                                # 如果没有识别到货币，使用默认阈值
                                threshold = self.CURRENCY_THRESHOLDS.get('default', 5.0)

                            if amount < threshold:
                                currency_display = currency if currency else "默认货币"
                                print(f"检测到红包金额 {amount}{currency_display} < 阈值 {threshold}{currency_display}，跳过处理: {text[:50]}...")
                                continue
                            else:
                                currency_display = currency if currency else "默认货币"
                                print(f"检测到红包金额 {amount}{currency_display} >= 阈值 {threshold}{currency_display}，继续处理")
                        else:
                            print(f"无法提取红包金额，继续处理: {text[:50]}...")
                    # =======================================================

                    # 检查消息是否包含关键词
                    if any(keyword in text for keyword in self.KEYWORDS):
                        print(f"检测到关键词消息: {text[:50]}...")
                        await self.forward_message(message)

                        # 更新处理时间
                        self.last_processed_time = time.time()
                        self.message_counter += 1

                except Exception as e:
                    print(f"消息处理出错: {str(e)}")

                # 计算处理时间
                process_time = time.time() - start_time
                if process_time > 0.5:  # 如果处理时间超过0.5秒
                    print(f"警告: 消息处理耗时 {process_time:.2f} 秒")

            # 短暂休眠以避免CPU过度占用
            await asyncio.sleep(0.01)

            # 每分钟报告一次性能
            if time.time() - self.start_time > 60:
                avg_time = (time.time() - self.start_time) / max(1, self.message_counter)
                print(f"性能报告: 已处理 {self.message_counter} 条消息，平均延迟 {avg_time:.2f} 秒")
                self.start_time = time.time()
                self.message_counter = 0

    async def message_handler(self, event):
        """接收新消息并加入队列"""
        # 将消息加入处理队列
        self.message_queue.append(event)

        # 如果队列长度超过阈值，打印警告
        if len(self.message_queue) > 10:
            print(f"警告: 美女在排队，当前排队长度: {len(self.message_queue)}")

    async def start(self):
        """启动服务"""
        try:
            await self._ensure_connection()

            # 打印登录信息
            me = await self.user_client.get_me()
            print(f"用户账号: {me.first_name} (ID: {me.id})")

            # 测试机器人连接
            try:
                bot_info = await self.bot.get_me()
                print(f"机器人账号: @{bot_info.username}")
            except Exception as e:
                print(f"机器人连接失败: {str(e)}")
                print("请检查BOT_TOKEN是否正确")

            # 注册消息处理器 - 监听所有新消息
            self.user_client.add_event_handler(
                self.message_handler,
                events.NewMessage(incoming=True, outgoing=False)
            )

            # 启动消息处理队列任务
            asyncio.create_task(self.process_message_queue())

            print("服务已启动，开始监听所有消息...")
            await self.user_client.run_until_disconnected()

        except Exception as e:
            print(f"启动失败: {str(e)}")
        finally:
            await self.stop()

    async def stop(self):
        """停止服务"""
        try:
            if self.user_client.is_connected():
                await self.user_client.disconnect()
            print("服务已停止")
        except Exception as e:
            print(f"停止服务时出错: {str(e)}")


async def main():
    forwarder = TelegramForwarder()
    try:
        await forwarder.start()
    except KeyboardInterrupt:
        print("用户中断操作")
    except Exception as e:
        print(f"程序错误: {str(e)}")
    finally:
        await forwarder.stop()


if __name__ == '__main__':
    asyncio.run(main())
