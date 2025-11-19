"""
Universal Conversation Splitter v2.0 - JSON-Driven Engine
---------------------------------------------------------
By Tri-Octa Harvest Designs (2025)

A modular, JSON-configured conversation splitter that can handle
multiple platforms (ChatGPT, Claude, Discord, etc.) using data-driven rules.
"""

import json
import os
from datetime import datetime
from pathlib import Path
import re


class UniversalSplitterEngine:
    """Main engine that processes conversations using JSON-driven configs."""
    
    def __init__(self):
        self.configs_dir = Path("configs")
        self.templates_dir = Path("templates")
        self.platform_detection = {}
        self.parser_configs = {}
        self.output_formats = {}
        self.load_all_configs()
    
    def load_all_configs(self):
        """Load all JSON configuration files."""
        try:
            # Load platform detection rules
            detection_file = self.configs_dir / "platform_detection.json"
            if detection_file.exists():
                with open(detection_file, 'r', encoding='utf-8') as f:
                    self.platform_detection = json.load(f)
            
            # Load parser configs for each platform
            for config_file in self.configs_dir.glob("*_parser.json"):
                platform = config_file.stem.replace("_parser", "")
                with open(config_file, 'r', encoding='utf-8') as f:
                    self.parser_configs[platform] = json.load(f)
            
            # Load output format templates
            output_file = self.configs_dir / "output_formats.json"
            if output_file.exists():
                with open(output_file, 'r', encoding='utf-8') as f:
                    self.output_formats = json.load(f)
                    
            print(f"✓ Loaded configs for platforms: {list(self.parser_configs.keys())}")
            
        except Exception as e:
            print(f"⚠️ Error loading configs: {e}")
            print("Using minimal fallback configuration...")
            self.setup_fallback_config()
    
    def setup_fallback_config(self):
        """Setup minimal config if JSON files are missing."""
        self.platform_detection = {
            "chatgpt": {
                "required_keys": ["mapping"],
                "optional_keys": ["title", "create_time"]
            }
        }
        
        self.parser_configs = {
            "chatgpt": {
                "conversations_path": "",
                "message_container": "mapping", 
                "title_path": "title",
                "author_path": "message.author.role",
                "content_path": "message.content.parts",
                "timestamp_path": "message.create_time",
                "timestamp_format": "unix",
                "user_role": "user",
                "ai_role": "assistant"
            }
        }
    
    def detect_platform(self, data):
        """Auto-detect which platform the conversation data is from."""
        print("🔍 Detecting conversation platform...")
        
        # Handle different root data structures
        if isinstance(data, str):
            # HTML content - check for Gemini patterns
            if ("Gemini Apps" in data and "Prompted " in data) or ("mdl-cell" in data and "content-cell" in data):
                print("✓ Detected platform: GEMINI")
                return "gemini"
            
            print("⚠️ HTML content found but couldn't identify platform - defaulting to ChatGPT")
            return "chatgpt"
            
        elif isinstance(data, list):
            # If it's a list, check the first item for platform detection
            if data and isinstance(data[0], dict):
                sample_data = data[0]
            else:
                print("⚠️ Could not analyze list structure - defaulting to ChatGPT")
                return "chatgpt"
        else:
            sample_data = data
        
        # Skip non-platform keys in platform_detection
        platform_keys = [key for key in self.platform_detection.keys() 
                         if key not in ["detection_order", "fallback_platform", "detection_settings"]]
        
        detection_order = self.platform_detection.get("detection_order", platform_keys)
        
        for platform in detection_order:
            if platform == "gemini":  # Skip Gemini for JSON data
                continue
            if platform in platform_keys:
                rules = self.platform_detection.get(platform, {})
                if self.matches_platform_signature(sample_data, rules):
                    print(f"✓ Detected platform: {platform.upper()}")
                    return platform
        
        print("⚠️ Could not detect platform - defaulting to ChatGPT")
        return self.platform_detection.get("fallback_platform", "chatgpt")
    
    def matches_platform_signature(self, data, rules):
        """Check if data matches platform signature rules."""
        # Skip file-based platforms for JSON data
        if "file_patterns" in rules and isinstance(data, dict):
            return False
        
        # Check for required keys
        required_keys = rules.get("required_keys", [])
        if required_keys:  # If there are required keys, at least one must exist
            for key in required_keys:
                if not self.has_nested_key(data, key):
                    return False
        
        # Check structure patterns if specified
        if "structure_pattern" in rules:
            return self.matches_structure_pattern(data, rules["structure_pattern"])
        
        return True
    
    def has_nested_key(self, data, key_path):
        """Check if nested key exists (supports dot notation like 'mapping.message')."""
        keys = key_path.split('.')
        current = data
        
        try:
            for key in keys:
                if isinstance(current, dict) and key in current:
                    current = current[key]
                else:
                    return False
            return True
        except:
            return False
    
    def extract_conversations(self, data, platform):
        """Extract individual conversations using platform-specific rules."""
        if platform not in self.parser_configs:
            raise ValueError(f"No parser config found for platform: {platform}")
        
        config = self.parser_configs[platform]
        conversations = []
        
        if platform == "chatgpt":
            conversations = self.extract_chatgpt_conversations(data, config)
        elif platform == "claude":
            conversations = self.extract_claude_conversations(data, config)
        elif platform == "gemini":
            conversations = self.extract_gemini_conversations(data, config)
        else:
            # Future: Add other platform extractors
            print(f"⚠️ Parser for {platform} not implemented yet")
        
        return conversations
    
    def extract_gemini_conversations(self, data, config):
        """Extract Gemini conversations from HTML export using proper HTML parsing."""
        conversations = []
        
        # For HTML content, data would be the raw HTML string
        if not isinstance(data, str):
            print("⚠️ Gemini parser expects HTML string data")
            return conversations
        
        # Find all conversation blocks using a more robust approach
        conversation_blocks = self.find_conversation_blocks(data)
        print(f"📋 Processing {len(conversation_blocks)} conversation blocks...")
        
        for idx, block in enumerate(conversation_blocks, 1):
            try:
                conversation = self.parse_gemini_html_block(block, config, idx)
                if conversation:
                    conversations.append(conversation)
                else:
                    print(f"⚠️ Conversation {idx}: No content extracted")
            except Exception as e:
                print(f"⚠️ Error parsing Gemini conversation {idx}: {e}")
        
        print(f"✅ Successfully extracted {len(conversations)} conversations")
        return conversations
    
    def find_conversation_blocks(self, html_content):
        """Find all conversation blocks in the Gemini HTML export using robust splitting."""
        import re
        
        # Use regex to find all conversation blocks
        # Each block starts with the outer-cell div and contains the conversation
        pattern = r'<div class="outer-cell mdl-cell mdl-cell--12-col mdl-shadow--2dp">(.*?)</div></div></div>'
        
        # Find all matches (conversations)
        matches = re.findall(pattern, html_content, re.DOTALL)
        
        # Reconstruct full blocks by adding back the wrapper div
        full_blocks = []
        for match in matches:
            full_block = f'<div class="outer-cell mdl-cell mdl-cell--12-col mdl-shadow--2dp">{match}</div></div></div>'
            full_blocks.append(full_block)
        
        print(f"🔍 Found {len(full_blocks)} conversation blocks in HTML")
        return full_blocks
    
    def debug_line_content(self, lines, index):
        """Debug function to examine why 'Prompted' detection is failing."""
        if index > 3:
            return
            
        print(f"🔍 ULTRA DEBUG {index}: Line analysis")
        for i, line in enumerate(lines[:3]):
            print(f"   Line {i+1} full content: '{line}'")
            print(f"   Line {i+1} length: {len(line)}")
            print(f"   Line {i+1} startswith 'Prompted ': {line.startswith('Prompted ')}")
            print(f"   Line {i+1} first 10 chars repr: {repr(line[:10])}")
            print(f"   Line {i+1} first word: '{line.split()[0] if line.split() else 'EMPTY'}'")
            print("   ---")
    
    def parse_gemini_html_block(self, html_block, config, index):
        """Parse a single Gemini conversation block from HTML."""
        
        # Debug: Check if we can find the content cell
        content_start = html_block.find('<div class="content-cell mdl-cell mdl-cell--6-col mdl-typography--body-1">')
        if content_start == -1:
            if index <= 5:  # Only debug first few failures
                print(f"🔍 Debug {index}: No content-cell found")
            return None
        
        content_end = html_block.find('</div>', content_start)
        if content_end == -1:
            content_end = len(html_block)
        
        # Extract just the content section
        content_section = html_block[content_start:content_end]
        
        # Convert HTML to text
        content_text = self.extract_text_from_html(content_section)
        
        # Split into lines and clean
        lines = [line.strip() for line in content_text.split('\n') if line.strip()]
        
        if not lines:
            if index <= 5:
                print(f"🔍 Debug {index}: No lines extracted from content")
            return None
        
        # Ultra debug to see exact line content
        self.debug_line_content(lines, index)
        
        # Debug: Print first few lines for troubleshooting
        if index <= 3:
            print(f"🔍 Debug {index}: First 3 lines extracted:")
            for i, line in enumerate(lines[:3]):
                print(f"   Line {i+1}: '{line[:100]}...'")
        
        # Find the user message (starts with "Prompted " or "Created ")
        user_content = None
        timestamp = None
        ai_content_lines = []
        
        found_user_message = False
        found_timestamp = False
        
        for i, line in enumerate(lines):
            # Check for user message patterns (handle both regular space and non-breaking space)
            if line.startswith("Prompted ") or line.startswith("Prompted\xa0"):
                # Handle both regular space and non-breaking space
                if line.startswith("Prompted "):
                    user_content = line[9:].strip()  # Remove "Prompted "
                else:
                    user_content = line[9:].strip()  # Remove "Prompted\xa0" (same length)
                found_user_message = True
                if index <= 3:
                    print(f"🔍 Debug {index}: Found 'Prompted' - extracted: '{user_content[:50]}...'")
            elif line.startswith("Created ") or line.startswith("Created\xa0"):
                # Handle both regular space and non-breaking space
                if line.startswith("Created "):
                    user_content = line[8:].strip()  # Remove "Created "
                else:
                    user_content = line[8:].strip()  # Remove "Created\xa0" (same length)
                found_user_message = True
                if index <= 3:
                    print(f"🔍 Debug {index}: Found 'Created' - extracted: '{user_content[:50]}...'")
            
            # Check for timestamp after user message
            elif found_user_message and not found_timestamp and self.is_gemini_timestamp(line):
                timestamp = line
                found_timestamp = True
                if index <= 3:
                    print(f"🔍 Debug {index}: Found timestamp: '{timestamp}'")
            
            # Collect AI response after timestamp
            elif found_user_message:
                # Skip empty lines and stop at footer
                if line and not line.startswith("Products:") and not line.startswith("Why is this here?"):
                    if not self.is_gemini_timestamp(line):  # Don't include timestamps in AI content
                        ai_content_lines.append(line)
        
        # Debug: Check what we found
        if index <= 3:
            print(f"🔍 Debug {index}: User content: '{user_content[:50] if user_content else 'None'}...'")
            print(f"🔍 Debug {index}: AI lines count: {len(ai_content_lines)}")
        
        # Build the conversation
        if not user_content:
            if index <= 5:
                print(f"🔍 Debug {index}: No user content found (no 'Prompted' or 'Created')")
            return None
        
        ai_content = "\n".join(ai_content_lines).strip()
        
        messages = []
        
        # Add user message
        messages.append({
            "author": "user",
            "content": [user_content],
            "timestamp": timestamp,
            "is_user": True
        })
        
        # Add AI message if it exists
        if ai_content:
            messages.append({
                "author": "assistant",
                "content": [ai_content],
                "timestamp": timestamp,
                "is_user": False
            })
        
        # Generate title from user content
        title_words = user_content.split()[:8]
        title = " ".join(title_words)
        if len(title_words) >= 8:
            title += "..."
        
        if index <= 3:
            print(f"🔍 Debug {index}: Success! Title: '{title}'")
        
        return {
            "title": title,
            "messages": messages,
            "index": index,
            "platform": "gemini"
        }
    def extract_text_from_html(self, html_content):
        """Better HTML to text extraction with proper entity decoding and paragraph preservation."""
        import re
        
        # First, handle paragraph tags by converting them to line breaks
        text = re.sub(r'<p[^>]*>', '\n\n', html_content)
        text = re.sub(r'</p>', '', text)
        
        # Handle other block elements
        text = re.sub(r'<br\s*/?>', '\n', text)
        text = re.sub(r'<div[^>]*>', '\n', text)
        text = re.sub(r'</div>', '', text)
        
        # Handle list items
        text = re.sub(r'<li[^>]*>', '\n• ', text)
        text = re.sub(r'</li>', '', text)
        
        # Handle headers
        text = re.sub(r'<h[1-6][^>]*>', '\n## ', text)
        text = re.sub(r'</h[1-6]>', '\n', text)
        
        # Remove all remaining HTML tags
        text = re.sub(r'<[^<]+?>', '', text)
        
        # Decode HTML entities properly
        html_entities = {
            '&nbsp;': ' ',
            '&amp;': '&',
            '&lt;': '<',
            '&gt;': '>',
            '&quot;': '"',
            '&#39;': "'",
            '&#x27;': "'",
            '&apos;': "'",
            '&rsquo;': "'",
            '&lsquo;': "'",
            '&rdquo;': '"',
            '&ldquo;': '"',
            '&mdash;': '—',
            '&ndash;': '–',
            '&hellip;': '…'
        }
        
        for entity, replacement in html_entities.items():
            text = text.replace(entity, replacement)
        
        # Clean up excessive whitespace while preserving intentional breaks
        text = re.sub(r'\n\s*\n\s*\n', '\n\n', text)  # Max 2 consecutive newlines
        text = re.sub(r'[ \t]+', ' ', text)  # Multiple spaces/tabs to single space
        text = re.sub(r'\n ', '\n', text)  # Remove leading spaces after newlines
        
        return text.strip()
    
    def is_gemini_timestamp(self, text):
        """Check if a line looks like a Gemini timestamp."""
        # Pattern: "Nov 11, 2025, 9:04:17 PM PST"
        import re
        pattern = r'^[A-Za-z]{3} \d{1,2}, \d{4}, \d{1,2}:\d{2}:\d{2} [AP]M [A-Z]{3}$'
        return bool(re.match(pattern, text.strip()))
    
    def extract_claude_conversations(self, data, config):
        """Extract Claude conversations using JSON config."""
        conversations = []
        
        # Handle different data structures - Claude can export as single conversation or list
        if isinstance(data, list):
            raw_conversations = data
        elif "conversations" in data:
            raw_conversations = data["conversations"]
        elif "chat_messages" in data:
            # Single conversation format
            raw_conversations = [data]
        else:
            raw_conversations = [data]  # Try to parse as single conversation
        
        for idx, conv_data in enumerate(raw_conversations, 1):
            try:
                conversation = self.parse_claude_conversation(conv_data, config, idx)
                if conversation:
                    conversations.append(conversation)
            except Exception as e:
                print(f"⚠️ Error parsing Claude conversation {idx}: {e}")
        
        return conversations
    
    def parse_claude_conversation(self, conv_data, config, index):
        """Parse a single Claude conversation."""
        title = conv_data.get(config.get("title_path", "name"), f"Claude Conversation {index}")
        chat_messages = conv_data.get(config.get("message_container", "chat_messages"), [])
        
        messages = []
        for msg_data in chat_messages:
            try:
                # Extract message components using config paths
                sender = self.get_nested_value(msg_data, config.get("author_path", "sender"))
                content = self.get_nested_value(msg_data, config.get("content_path", "text"))
                timestamp = self.get_nested_value(msg_data, config.get("timestamp_path", "created_at"))
                
                if content and sender:
                    message = {
                        "author": sender,
                        "content": [content],  # Wrap in list for consistency
                        "timestamp": timestamp,
                        "is_user": sender == config.get("author_mapping", {}).get("user_roles", ["human"])[0]
                    }
                    messages.append(message)
            except Exception as e:
                continue
        
        return {
            "title": title,
            "messages": messages,
            "index": index,
            "platform": "claude"
        }
    
    def extract_chatgpt_conversations(self, data, config):
        """Extract ChatGPT conversations using JSON config."""
        conversations = []
        
        # Safety check - ChatGPT parser expects JSON data, not HTML strings
        if isinstance(data, str):
            print("⚠️ ChatGPT parser received string data instead of JSON - skipping")
            return conversations
        
        # Handle different data structures
        if isinstance(data, list):
            raw_conversations = data
        else:
            raw_conversations = data.get("conversations", data.get("items", [data]))
        
        for idx, conv_data in enumerate(raw_conversations, 1):
            try:
                conversation = self.parse_chatgpt_conversation(conv_data, config, idx)
                if conversation:
                    conversations.append(conversation)
            except Exception as e:
                print(f"⚠️ Error parsing conversation {idx}: {e}")
        
        return conversations
    
    def parse_chatgpt_conversation(self, conv_data, config, index):
        """Parse a single ChatGPT conversation."""
        title = conv_data.get(config.get("title_path", "title"), f"Conversation {index}")
        mapping = conv_data.get(config.get("message_container", "mapping"), {})
        
        messages = []
        for msg_id, msg_wrapper in mapping.items():
            try:
                msg_data = msg_wrapper.get("message")
                if not msg_data:
                    continue
                
                # Extract message components using config paths
                author_role = self.get_nested_value(msg_data, config.get("author_path", "author.role"))
                content_parts = self.get_nested_value(msg_data, config.get("content_path", "content.parts"))
                timestamp = self.get_nested_value(msg_data, config.get("timestamp_path", "create_time"))
                
                if content_parts and author_role:
                    message = {
                        "author": author_role,
                        "content": content_parts,
                        "timestamp": timestamp,
                        "is_user": author_role == config.get("user_role", "user")
                    }
                    messages.append(message)
            except Exception as e:
                continue
        
        return {
            "title": title,
            "messages": messages,
            "index": index,
            "platform": "chatgpt"
        }
    
    def get_nested_value(self, data, path):
        """Get value from nested dict using dot notation."""
        if not path or not data:
            return None
        
        keys = path.split('.')
        current = data
        
        try:
            for key in keys:
                current = current[key]
            return current
        except (KeyError, TypeError):
            return None
    
    def sanitize_filename(self, name):
        """Clean filename for cross-platform compatibility."""
        forbidden = ['<', '>', ':', '"', '/', '\\', '|', '?', '*']
        for char in forbidden:
            name = name.replace(char, "_")
        return name.strip()[:150]
    
    def format_timestamp(self, timestamp, format_type="unix"):
        """Convert timestamp to readable string."""
        if not timestamp:
            return "Unknown"
        
        try:
            if format_type == "unix":
                dt = datetime.fromtimestamp(timestamp)
            elif format_type == "iso8601":
                # Handle ISO8601 format like "2024-01-15T10:30:00.000Z"
                if isinstance(timestamp, str):
                    # Remove timezone and microseconds for simple parsing
                    clean_timestamp = timestamp.replace('Z', '').split('.')[0]
                    dt = datetime.fromisoformat(clean_timestamp)
                else:
                    dt = datetime.fromtimestamp(timestamp)
            else:
                # Fallback
                dt = datetime.fromtimestamp(timestamp)
            
            return dt.strftime("%Y-%m-%d %H:%M")
        except:
            return str(timestamp)[:16]  # Return truncated string if parsing fails
    
    def process_file(self, input_file, output_folder="ConversationExports", 
                    format_type="md", user_name="User", ai_name="Assistant"):
        """Main processing function - load, detect, parse, and export."""
        
        print("=" * 70)
        print("🚀 Universal Conversation Splitter v2.0")
        print("=" * 70)
        
        # Load input file
        print(f"📂 Loading: {input_file}")
        try:
            # Check if it's an HTML file
            if input_file.lower().endswith('.html'):
                with open(input_file, 'r', encoding='utf-8') as f:
                    data = f.read()
            else:
                with open(input_file, 'r', encoding='utf-8') as f:
                    data = json.load(f)
        except Exception as e:
            print(f"❌ Error loading file: {e}")
            return False
        
        # Detect platform
        platform = self.detect_platform(data)
        
        # Extract conversations
        print(f"📊 Extracting conversations...")
        conversations = self.extract_conversations(data, platform)
        
        if not conversations:
            print("❌ No conversations found!")
            return False
        
        print(f"✓ Found {len(conversations)} conversations")
        
        # Create output directory
        os.makedirs(output_folder, exist_ok=True)
        
        # Process each conversation
        return self.export_conversations(conversations, output_folder, format_type, 
                                       user_name, ai_name, platform)
    
    def export_conversations(self, conversations, output_folder, format_type, 
                           user_name, ai_name, platform):
        """Export conversations in the specified format."""
        
        print(f"\n📝 Exporting {len(conversations)} conversations as {format_type.upper()}...")
        
        success_count = 0
        
        for conversation in conversations:
            try:
                # Generate filename
                safe_title = self.sanitize_filename(conversation["title"])
                index = conversation["index"]
                filename = f"{index:03d} - {safe_title}.{format_type}"
                filepath = os.path.join(output_folder, filename)
                
                # Generate content based on format
                if format_type == "md":
                    content = self.generate_markdown(conversation, user_name, ai_name)
                elif format_type == "html":
                    content = self.generate_html(conversation, user_name, ai_name)
                elif format_type == "txt":
                    content = self.generate_text(conversation, user_name, ai_name)
                else:
                    print(f"⚠️ Unknown format: {format_type}")
                    continue
                
                # Write file
                with open(filepath, 'w', encoding='utf-8') as f:
                    f.write(content)
                
                print(f"   ✓ {filename}")
                success_count += 1
                
            except Exception as e:
                print(f"   ❌ Failed: {conversation['title']} - {e}")
        
        print(f"\n🎉 Export complete! {success_count}/{len(conversations)} files created")
        print(f"📁 Output folder: {output_folder}")
        return True
    
    def generate_markdown(self, conversation, user_name, ai_name):
        """Generate Markdown content for a conversation."""
        title = conversation["title"]
        messages = conversation["messages"]
        platform = conversation.get('platform', 'unknown')
        
        # Header with frontmatter
        content = f"""---
title: {title}
date: {datetime.now().strftime('%Y-%m-%d')}
user: {user_name}
ai: {ai_name}
platform: {platform}
---

# {title}

"""
        
        # Messages
        for message in messages:
            author = user_name if message["is_user"] else ai_name
            
            # Handle different timestamp formats by platform
            if platform == "claude":
                timestamp = self.format_timestamp(message["timestamp"], "iso8601")
            elif platform == "gemini":
                # Gemini timestamps are already formatted strings like "Oct 15, 2025, 4:31:19 PM PST"
                timestamp = message.get("timestamp", "Unknown")
            else:
                timestamp = self.format_timestamp(message["timestamp"], "unix")
            
            content += f"## {author} — {timestamp}\n\n"
            
            for part in message["content"]:
                if isinstance(part, str) and part.strip():
                    # Clean up the content for better display
                    clean_content = part.strip()
                    
                    # For Gemini content, ensure proper paragraph spacing
                    if platform == "gemini":
                        # Split by double newlines and rejoin to ensure consistent paragraph breaks
                        paragraphs = [p.strip() for p in clean_content.split('\n\n') if p.strip()]
                        clean_content = '\n\n'.join(paragraphs)
                    
                    content += clean_content + "\n\n"
        
        # Footer
        content += "---\n\n"
        content += "> Exported with **Universal Conversation Splitter v2.0**\n"
        content += "> © 2025 Tri-Octa Harvest Designs\n"
        
        return content
    
    def generate_html(self, conversation, user_name, ai_name):
        """Generate HTML content for a conversation."""
        title = conversation["title"]
        messages = conversation["messages"]
        
        # Build message blocks
        message_blocks = []
        for message in messages:
            author = user_name if message["is_user"] else ai_name
            timestamp = self.format_timestamp(message["timestamp"])
            
            if message["is_user"]:
                style = "background: #f8f9fa; border-left: 4px solid #007acc; color: #007acc;"
            else:
                style = "background: #f0f8f0; border-left: 4px solid #28a745; color: #28a745;"
            
            content_html = ""
            for part in message["content"]:
                if isinstance(part, str) and part.strip():
                    # Basic HTML escaping and line breaks
                    clean_part = part.replace('&', '&amp;').replace('<', '&lt;').replace('>', '&gt;')
                    clean_part = clean_part.replace('\n', '<br>')
                    content_html += f"<p>{clean_part}</p>"
            
            message_block = f'''<div style="margin: 20px 0; padding: 15px; border-radius: 5px; {style}">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
        <strong style="font-size: 1.1em;">{author}</strong>
        <span style="color: #666; font-size: 0.9em;">{timestamp}</span>
    </div>
    <div style="color: #333;">
        {content_html}
    </div>
</div>'''
            message_blocks.append(message_block)
        
        # Full HTML document
        html_content = f'''<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>{title}</title>
    <style>
        body {{
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 900px;
            margin: 0 auto;
            padding: 40px 20px;
            background: white;
        }}
        h1 {{
            color: #2c3e50;
            border-bottom: 3px solid #3498db;
            padding-bottom: 15px;
            margin-bottom: 30px;
        }}
        .header {{
            background: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 30px;
            border-left: 4px solid #17a2b8;
        }}
    </style>
</head>
<body>
    <h1>{title}</h1>
    <div class="header">
        <strong>Participants:</strong> {user_name} & {ai_name}<br>
        <strong>Platform:</strong> {conversation['platform'].title()}<br>
        <strong>Exported:</strong> {datetime.now().strftime('%Y-%m-%d %H:%M')}
    </div>
    
    {"".join(message_blocks)}
    
    <div style="margin-top: 50px; padding: 20px; background: #f8f9fa; border-radius: 5px; border-left: 4px solid #17a2b8;">
        <strong>💡 Pro Tip:</strong> Press <kbd>Ctrl+P</kbd> and select "Save as PDF" to create a permanent copy.
    </div>
</body>
</html>'''
        
        return html_content
    
    def generate_text(self, conversation, user_name, ai_name):
        """Generate plain text content for a conversation."""
        title = conversation["title"]
        messages = conversation["messages"]
        
        content = f"Conversation: {title}\n"
        content += f"Participants: {user_name} & {ai_name}\n"
        content += f"Platform: {conversation['platform'].title()}\n"
        content += f"Exported: {datetime.now().strftime('%Y-%m-%d %H:%M')}\n\n"
        content += "=" * 60 + "\n\n"
        
        for message in messages:
            author = user_name if message["is_user"] else ai_name
            timestamp = self.format_timestamp(message["timestamp"])
            
            content += f"{author} ({timestamp}):\n"
            for part in message["content"]:
                if isinstance(part, str) and part.strip():
                    content += part.strip() + "\n"
            content += "\n" + "-" * 40 + "\n\n"
        
        return content


def main():
    """Interactive main function."""
    engine = UniversalSplitterEngine()
    
    # Get user input
    input_file = input("Enter conversation file (default: conversations.json): ").strip() or "conversations.json"
    
    if not os.path.exists(input_file):
        print(f"❌ File '{input_file}' not found!")
        print("💡 Supported formats:")
        print("   • ChatGPT/Claude: conversations.json")
        print("   • Gemini: *.html (from Google Takeout)")
        return
    
    output_folder = input("Enter output folder (default: ConversationExports): ").strip() or "ConversationExports"
    
    print("\nChoose format:")
    print("  [1] Markdown (.md)")
    print("  [2] HTML (.html)")  
    print("  [3] Plain Text (.txt)")
    format_choice = input("Enter choice (1/2/3, default: 1): ").strip()
    
    format_map = {"2": "html", "3": "txt"}
    format_type = format_map.get(format_choice, "md")
    
    user_name = input("\nEnter your name (default: User): ").strip() or "User"
    ai_name = input("Enter AI name (default: Assistant): ").strip() or "Assistant"
    
    # Process the file
    success = engine.process_file(input_file, output_folder, format_type, user_name, ai_name)
    
    if success:
        print(f"\n✨ All done! Check the '{output_folder}' folder for your conversations.")
    
    input("\nPress Enter to exit...")


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n\n❌ Cancelled by user.")
    except Exception as e:
        print(f"\n❌ Unexpected error: {e}")
        input("Press Enter to exit...")
